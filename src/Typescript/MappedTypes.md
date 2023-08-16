# Отображения

### Pick
Оставить только выбранные свойства

```ts
interface Person {
    name: string,
    age: number,
    address: string,
}

type PersonNameAddress<T, K> = Pick<Person, 'name' | 'address'>;
const three: PersonNameAddress<Person, 'name' | 'address'> = {name: 'we', address:'asd'};
```

### Readonly
Сделать все свойста readonly

```ts
interface Person {
    name: string,
    age: number,
    address: string,
}

const four: Readonly<Person> = {name: 'we', address:'asd', age: 11};
four.age = 12; // error
```

### Partial
Сделать все свойства необязательными
```ts
interface Person {
    name: string,
    age: number,
    address: string,
}

const four: Partial<Person> = {name: 'we', address:'asd'};
```

### Required
Сделать все свойства обязательными
```ts
interface Person {
    name: string,
    age: number,
    address?: string,
}

const four: Required<Person> = {name: 'we', age: 12}; //error
```

### Комбинация нескольких типов
Все свойства не обязательные и только для чтения
```ts
const four: Readonly<Partial<Person>> = {name: 'we', address:'asd'};
```