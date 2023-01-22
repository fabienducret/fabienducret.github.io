---
title: "L'inversion de dépendance"
date: 2023-01-22T14:23:14+01:00
draft: false
---

L'inversion des dépendances est un des principes clé de programmation. C'est le fameux **D** de l'acronyme **SOLID**.

Mais en quoi consiste ce principe et pourquoi est-il si utile ?

---

## Le couplage, fléau de la maintenabilité et de la testabilité

Lorsque plusieurs modules sont fortement couplés, la maintenabilité ainsi que la testabilité deviennent plus difficiles.

Prenons un exemple concret avec la récupération d'élements depuis une base de données :

```ts
import { MySQLConnection } from './mysql.repository';

class BookService {
  private readonly connection: MySQLConnection;

  constructor() {
    this.connection = new MySQLConnection();
  }

  getById(id: string) {
    return this.connection.getBook(id);
  }
}
```

Dans cet exemple la classe **BookService** est fortement liée à **MySQLConnection**.

Voici les effets de ce couplage :

- un test unitaire sur la méthode **getById** appellera directement la base de données. On souhaite l'éviter pour des raisons de performance et de déterminisme des tests.
- si on souhaite changer de système de base de données, alors la classe **BookService** devra être modifiée.

Le sens du couplage entre ces deux classes peut être représenté de cette manière :

```
BookService -> MySQLConnection
```

---

## L'inversion de dépendance

Comme son nom l'indique, **l'inversion des dépendances permet d'inverser le sens du couplage entre un module de haut niveau (notre service) et un détail d'implémentation (notre connexion MySQL)**.

Le but étant que la classe **BookService** ne dépende plus de l'implémentation de la connexion MySQL.

Voici comment mettre en place ce principe :

```ts
export interface Repository {
  getBook(id: string): BookFromRepository;
}

class BookService {
  constructor(private readonly repository: Repository) {}

  getById(id: string) {
    return this.repository.getBook(id);
  }
}
```

On injecte un type **Repository** directement dans le constructeur de la classe **BookService**. Ainsi, la classe n'a pas connaissance de l'implémentation qui lui sera injectée. Elle sait simplement que cette implémentation respectera l'interface **Repository**.

La classe **MySQLConnection** doit maintenant implémentée l'interface **Repository** :

```ts
class MySQLConnection implements Repository {
  getBook(id: string) {
    // implémentation concrète
  }
}
```

Il n'y a plus qu'à composer avec ces deux classes au lancement de l'application :

```ts
const main = () => {
  const repository = new MySQLConnection();
  const bookService = new BookService(repository);
  // ...
};
```

Voici le nouveau sens du couplage :

```
BookService -> Repository | <- MySQLConnection
```

**BookService dépend dorénavant d'une interface et non d'une implémentation.**

De plus il est maintenant possible de tester la classe **BookService** en passant par une implémentation **fake** qui n'est pas connectée à une base de données.

```ts
test('getById', () => {
  const repository = new FakeRepository();
  const bookService = new BookService(repository);
  // suite du test
});
```
