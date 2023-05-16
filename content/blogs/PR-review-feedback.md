---
title: "PR Review Feedback"
date: 2023-05-16T09:25:59+08:00
author: "Feng Xue"
draft: false
tags: ["PR review", "Summary"]
toc: true
---

I do not have a good memory and cannot remember all the mistakes I made in the previous PR reviewing. Since `The palest ink is better than the best memory`, so let me summarize the mistakes I have made here to reminder myself, hope it may help.

## React

### Necessary of `useEffect`?

When we use the `useEffect`, we need to think about if it's necessary, especially for the initial mount effect. Instead of updating some values based on the dependency of `useEffect`, we should check if it's possible to set the value initially directly.

## React-query

### Use the query data as source of data, instead of triggering/update from UI

One example would be undoing the editing to a table item with Material UI's `DataGrid`. The procedure of undo is

1. Edit the table cell text with Material UI's editable function (Imperative)
2. Trigger the undo by clicking somewhere undo button/link (Declarative)

Here we have two ways to trigger the undo:

1. use Material UI's method to recover the data in the table and then the data would be updated sequentially

  ```js
  undoCallback: async () => {
  // Undo the process should be fast with update only one field value
  setUndoStateMap((value) => ({ ...value, [newRow.id]: key }))
  // during the undo, use data grid's way to update the cell value manually
  await tableApiRef.current.startCellEditMode({ id: newRow.id, field: key })
  await tableApiRef.current.setEditCellValue({
      id: newRow.id,
      field: key,
      value: oldValue,
  })
  // If the cell is in edit mode, stop it
  if 
      await tableApiRef.current.stopCellEditMode({ id: newRow.id, field: key })
  }
  ```

2. update the table cell data directly and it would be reflected by React automatically

  ```js
  undoCallback: async () => {
    updateKnowTheDriverMutation.mutate({
      vehicleId: newRow.vehicleId,
      category: key,
      score: oldValue,
    })
  },
  ```

From the code we can see the second one is better since it's simpler and meets the react's principle `declarative` way. The imperative way needs to know each step of whole undo processing and it may contribute the mistake.

### Think of the rollback situation when using mutation

When we mutate the data, we need to think in this way: `if the action fails, do we need to rollback the previous value?` Here are the example codes about mutation:

```js
onMutate: async (variables) => {
  const knowTheDriverQuery = knowTheDriverListQuery()

  // Cancel any outgoing refetches (so they don't overwrite our optimistic update)
  await queryClient.cancelQueries(knowTheDriverQuery)

  const previousQueryData = knowTheDriverQuery.getData(queryClient)

  if (previousQueryData) {
    knowTheDriverQuery.setData(queryClient, {
      updater: updateFirstWhere(
        previousQueryData,
        (c) => c.vehicleId === variables.vehicleId,
        // Explicit return type so that we don't add properties by mistake
        (c): (typeof previousQueryData)[number] => ({
          ...c,
          [variables.category]: Number(variables.score),
        }),
      ),
    })
  }

  return { previousQueryData }
},
onError: (err, _variables, context) => {
  if (context?.previousQueryData) {
    knowTheDriverListQuery().setData(queryClient, {
      updater: context.previousQueryData,
    })
  }
  makeMutationErrorHandlerWithToast().onError(err)
}
```

## Date

### Send Js native Date to server, it would be serialized automatically and use this as the format in API

When we need to send the `date` to server, no matter what library we use to handle it, Luxon, Moment, etc, we should save the native `Date` to sever since it's native supported and could be used in all the browsers without supporting from third-part libraries.

## Typescript

### Parse data from external sources

When we get some data which may be arbitrary from external source, we can use [`zod`](https://zod.dev/) to parse the value, because Typescript cannot check the values in the runtime.

For example, we have a field about filters' initial values in the response of api, and it should have static options:

* vehicle_group_init,
* department_init,
* sub_department_init,
* drivers_init,
* event_type_init,

```js
export const apiCoachingDashboardFilterType = z.enum([
  'vehicle_group_init',
  'department_init',
  'sub_department_init',
  'drivers_init',
  'event_type_init',
])
```

In frontend, we cannot confirm the values sent from backend are correct. So before converting the values, we need to check if they meet our requirements

```js
export declare namespace CoachingDashboardFilterType {
  type ApiInput = {
    id: z.infer<typeof apiCoachingDashboardFilterType>
    default: string
  }
  type ParsedEvent = {
    id: 'vehicleGroup' | 'department' | 'section' | 'driver' | 'eventType'
    value: string
  }
}

export function parseUserSettings() {
  // ...

  const coachingDashboardFiltersRaw: Array<CoachingDashboardFilterType.ApiInput> =
    settings.coaching_dashboard_filters

  const coachingDashboardFilters = coachingDashboardFiltersRaw
    .map((filter) => {
      const parsedResult = apiCoachingDashboardFilterType.safeParse(filter.id)
      if (!parsedResult.success) {
        return null
      }
      const parsed: CoachingDashboardFilterType.ParsedEvent = {
        id: parsedResult.data,
        value: `${filter.default}`,
      }
      return parsed
    })
    .filter(Re.isNot(Re.isNil))
  // ...
}
```

And when we return the parsed values, instead of returning it directly, we can define a variable as parsed value type. With this, if the input data have some different values in the future, typescript would report it.

```js
const parsed: CoachingDashboardFilterType.ParsedEvent = {
  id: parsedResult.data,
  value: `${filter.default}`,
}
return parsed
```
