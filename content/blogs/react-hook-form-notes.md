---
title: "Powerful type-checking of React-hook-form and Learning Notes"
date: 2023-01-28T16:05:09+08:00
author: "Feng Xue"
tags: ["Typescript", "Frontend", "React", "React hook form"]
toc: true
usePageBundles: false
draft: false 
---

The first time I touched the `React-hook-form` was in 2020 and it also was the first time I learned React's new concept Hooks. During that time we compared all the popular form libraries, `Formik`, `react-form`, `final form` and etc, but we finally chose the `React-hook-form` even though it was still under the development during that time. It is because it uses React latest concept Hooks. And React was our fundamental framework in my previous company, so we didn't need to think about the compatibility.

Now it comes to the version 7 and there are more hooks. And these days I began to use it in our project to create multiple forms. After discussing with my colleague Rodrigo, I had a better deeper understanding to `React-hook-form`'s built-in schema validation when develop with typescript. Before we go deep, let me clear some basic concepts.

## Controlled vs Uncontrolled component 

This is the concept introduced by the React and the difference of them is where you will keep the source of truth. For the controlled components, we will pass a React controlled value to the form element,, React will take care of the state and render the component whenever you change the value. But for uncontrolled components, the state is saved in the DOM, which means when you update the value of form element, React will not notify it. We have to manually extract the value when need (through React's ref). But the benefit of that is it will not render the component either.

Let me clear it with the simple codes:

### Controlled Component

```js
const ControlledComponent = () => {
  const [someValue, setSomeValue] = useState()
  const handleChange= (event) => setSomeValue(event.target.value)

  return <input value={someValue} onChange={handleChange} />
}
```

So this `Controlled Component` will monitor the input value and keep **pushing** the value to the element.
> A form element becomes "controlled" if you set its value via a prop. That's all.

NOTE: `checked` for **Radio and Checkbox**.

NOTE: it's fine to pass `onChange` to form element, only setting its value will make it controlled. (I used to think the property `onChange` or `onClick` decides if the component is controlled or uncontrolled as well)

### Uncontrolled Component

```js
const UncontrolledComponent = () => {
  const testRef = useRef()

  const handleButtonClick = () => {
    const value = testRef.current?.value;
    // do something to the value
  }

  return (
    <div>
        <input type="text" ref={testRef} />
        <button onClick={handleButtonClick} />
    </div>
  )
}
```

Since the value is not assigned to the element `input`, React would not know its value you typed. To get the value, we can use the `ref`.

## Register

So for uncontrolled component, React-hook-form will use `register` method to generate the related methods for the form element without the property `value`.

```js
const { onChange, onBlur, name, ref } = register('firstName'); 
        
<input 
  onChange={onChange} // assign onChange event 
  onBlur={onBlur} // assign onBlur event
  name={name} // assign name prop
  ref={ref} // assign ref prop
/>
// same as above
<input {...register('firstName')} />
```

But for Controlled components, like material UI, it's recommended to use `Controller`. Its property `render` will pass a parameter `field` containing `value` to the Controlled Component.

```jsx
<Controller
  control={control}
  name="test"
  render={({
    field: { onChange, onBlur, value, name, ref },
    fieldState: { invalid, isTouched, isDirty, error },
    formState,
  }) => (
    <Checkbox
      onBlur={onBlur} // notify when input is touched
      onChange={onChange} // send value to hook form
      checked={value}
      inputRef={ref}
    />
  )}
/>
```

## Typescript with Schema

OK, before we go further, let's check one important of react-hook-form's features - **schema validation**. This is not a new thing, all the form libraries support schema validation and this is a kind of standard. But with help of typescript, react-hook-form provides a very powerful type-safe checking. I can explain this with example later. Let's simply check the setup. Here we will use [Zod](https://github.com/colinhacks/zod), which seems the best choice currently according to [this article](https://egghead.io/blog/zod-vs-yup-vs-joi-vs-io-ts-for-creating-runtime-typescript-validation-schemas) 

```tsx
const schema = z.object({
  name: z.string(),
  age: z.number()
});

type Schema = z.infer<typeof schema>;

const App = () => {
  const { register, handleSubmit } = useForm<Schema>({ resolver: zodResolver(schema) });

  //...
}
```

## Type checking

React-hook-form provides very powerful type-checking, it also exports multiple types used with typescript generic type. This helps us a lot with good developer experience which is also one goal of this library's design and philosiphy.

To find out the power of type checking, let's compare a correct example to a tricky one. 

According to the official doc of `setValue`, it's possible to set value to an **unregistered** field. When we set value to an unregistered field, we will registered in this field as well.

```js
// you can setValue to a unregistered input
setValue('notRegisteredInput', 'value'); // ✅ prefer to be registered
```

### Setup schema and initial value

So go further, let's choose Material UI's component `Select` as an Controlled example. Before defining the component, setup the schema and initial value first.

```tsx
// Select Option array
const SelectValue = [
  { value: "10", label: "The entire place" },
  { value: "20", label: "A private room" },
  { value: "30", label: "A shared room" },
] as const;

// define the schema and type through Zod
type Property = typeof SelectValue[number]["value"];
const VALUES: [Property, ...Property[]] = [
  SelectValue[0].value,
  ...SelectValue.slice(1).map((p) => p.value),
];

const PropertySchema = z.enum(VALUES);

// the Zod schema used for resolver in form
const formSchema = z.object({ example: PropertySchema });

type Inputs = z.infer<typeof formSchema>;

// default values provided to form
const initValues: Inputs = { example: "10" };
```

### Tricky Controlled Component

Then we can create a simple tricky component with Mui `Select`, which will not register the field in the form in advance, just pass the react-hook-form's types values `control`, `setValue` as parameters to allow the component to obtain and update the value.

```tsx
type Props<T extends FieldValues> = {
  name: FieldPath<T>;
  control: Control<T>;
  setFormValue: UseFormSetValue<T>;
  data: ReadonlyArray<{
    label: string;
    value: FieldPathValue<T, FieldPath<T>>;
  }>;
};

const Input1 = <T extends FieldValues>({
  name,
  control,
  setFormValue,
  data,
}: Props<T>) => {
  const value = useWatch({ name, control });

  const handleChange = (event: SelectChangeEvent) => setFormValue(
    name,
    (event.target.value + "11") as FieldPathValue<T, FieldPath<T>>
  );

  return (
    <FormControl fullWidth>
      <InputLabel id="demo-simple-select-label">Age</InputLabel>
      <Select
        labelId="demo-simple-select-label"
        id="demo-simple-select"
        value={value}
        label="Age"
        onChange={handleChange}
      >
        {data.map((entry) => {
          return (
            <MenuItem key={entry.value} value={entry.value}>
              {entry.label}
            </MenuItem>
          );
        })}
      </Select>
    </FormControl>
  );
};
```

For the input parameters, we use react-hook-form's types to let it check the types of name and values for us.

* `name: FieldPath<T>`
* `control: Control<T>`
* `setFormValue: UseFormSetValue<T>`
* `value: FieldPathValue<T, FieldPath<T>>`

And when the value changes, we will use the input method `setFormValue` to set the field value directly. But here we do a tricky thing, instead of using the value from the option, we manipulate it by appending a string `"11"`.

```tsx
const handleChange = (event: SelectChangeEvent) => {
  setFormValue(
    name,
    (event.target.value + "11") as FieldPathValue<T, FieldPath<T>>
  )
}
```

Let's use this component in the form:

```tsx
function App() {
  const methods = useForm<Inputs>({
    defaultValues: initValues,
    mode: "onChange",
    resolver: zodResolver(formSchema),
  });

  const {
    register,
    control,
    handleSubmit,
    setValue,
    formState: { errors },
  } = methods;

  const onSubmit: SubmitHandler<Inputs> = (data) => console.log(data, errors)

  return (
    <div className="App">
      <form onSubmit={handleSubmit(onSubmit)}>
        <Input1 name="example" setFormValue={setValue} control={control} data={SelectValue} />
        {errors.example && errors.example.message}
        <input type="submit" />
      </form>
    </div>
  );
}
```

We display the error if it exists. When we select the value in the `Select`, since the value has been manipulated, `Select` will be blank. But the error does not show either, only warning in the console. Definitely, this is not good, we do not have enough information about our error.


<div style="text-align: center; width: 500px;">
  <img src="/images/react-hooks/select-error-console.png">
</div>

### Correct Controlled Component

Let's create a correct controlled component and pass it wrong data, see how it gives our error message.

```tsx
const SelectValueWrong = [
  { value: "101", label: "The entire place" }, // This value is different from the one in type and schema
  { value: "20", label: "A private room" },
  { value: "30", label: "A shared room" },
] as const;

type Props3<T extends FieldValues> = {
  name: FieldPath<T>;
  control: Control<T>; // And we do not need to pass the method setValue since react-hook-form will handle it for us
  data: ReadonlyArray<{
    label: string;
    value: FieldPathValue<T, FieldPath<T>>;
  }>;
};

const Input3 = <T extends FieldValues>({ name, control, data }: Props3<T>) => (
  <Controller
    control={control}
    name={name}
    render={({ field }) => {
      return (
        <FormControl fullWidth>
          <InputLabel id="demo-simple-select-label">Age</InputLabel>
          <Select
            {...field}
            labelId="demo-simple-select-label"
            id="demo-simple-select"
            label="Age"
          >
            {data.map((entry) => {
              return (
                <MenuItem key={entry.value} value={entry.value}>
                  {entry.label}
                </MenuItem>
              );
            })}
          </Select>
        </FormControl>
      );
    }}
  />
);

// use it in the form
    <Input3 control={control} name="example3"
     data={SelectValueWrong} /> // pass the wrong select value
    {errors.example3 && errors.example3.message}
//...
```

When we select the wrong option, the value will still be displayed in the `Select`, but the schema validates the input value as well, set it as error and the error message is displayed.

<div style="text-align: center; width: 500px;">
  <img src="/images/react-hooks/select-error-message.png">
</div>

### Source codes

You can check all the codes here:

<iframe src="https://codesandbox.io/embed/proud-wildflower-sro8nz?fontsize=14&hidenavigation=1&theme=dark&view=preview"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="type-checking"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

## FormProvider and useFormContext? Nja

Another thought about `React-hook-form` is the `FormProvider` and `useFormContext`. When we pass the methods through them to the children components, we also lose the types of all the methods. Yes, of course, we can pass the type to `useFormContext`, but since we need to pass the form schema type twice, the type is still not 100% safe in theory. So if the form's elements are not deep, we would recommend to pass the `control` which would include the form's type to children components.
