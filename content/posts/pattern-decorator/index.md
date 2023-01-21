---
title: 'Le pattern Decorator pour gérer la complexité accidentelle'
date: 2023-01-21T17:49:53+01:00
draft: false
---

Il arrive parfois que certaines contraintes techniques polluent notre code. On appelle ça la **complexité accidentelle**.

Voici un exemple avec la gestion de la concurrence en **Go** .

---

Ici on définit une structure **Book** avec deux méthodes, une pour lire la propriété **name** et la deuxième pour l'écrire :

```go
type Book struct {
	name string
}

func (b *Book) Name() string {
	return b.name
}

func (b *Book) ChangeName(n string) {
	b.name = n
}
```

Si les deux méthodes sont exécutées en même temps dans un channel différent, l'application panique car elle ne sait pas gérer cette concurrence :

```
--- FAIL: TestMutex (0.00s)
    --- FAIL: TestMutex/Change_books's_name (0.05s)
        testing.go:1319: race detected during execution of test
    --- FAIL: TestMutex/Get_Book's_name (0.07s)
        testing.go:1319: race detected during execution of test
FAIL
FAIL    library 0.286s
FAIL
```

La solution la plus évidente serait d'ajouter la gestion de cette concurrence directement dans les méthodes de notre structure **Book** :

```go
type Book struct {
  sync.RWMutex
  name string
}

func (b *Book) Name() string {
  b.RLock()
  defer b.RUnlock()

  return b.name
}

func (b *Book) ChangeName(n string) {
  b.Lock()
  defer b.Unlock()

  b.name = n
}
```

Le problème de cette solution est qu'elle ajoute une complexité technique à notre structure. Ce n'est clairement pas idéal car on couple fortement notre structure avec cette notion de concurrence.

---

Le pattern **Decorator** peut nous aider pour décorreler cette complexité accidentelle de notre structure.

Voici une réponse possible à ce problème.

On passe par une interface **Book** qui sera implémentée par deux structures :

```go
type Book interface {
	Name() string
	ChangeName(i string)
}
```

L'implémentation par défaut, pure et sans logique externe :

```go
type DefaultBook struct {
	name string
}

func (b *DefaultBook) Name() string {
	return b.name
}

func (b *DefaultBook) ChangeName(n string) {
	b.name = n
}
```

Et voici l'implémentation qui a pour unique but de gérer la complexité accidentelle liée à la concurrence :

```go
type MutexBook struct {
	sync.RWMutex
	original DefaultBook
}

func (b *MutexBook) Name() string {
	b.RLock()
	defer b.RUnlock()

	return b.original.Name()
}

func (b *MutexBook) ChangeName(n string) {
	b.Lock()
	defer b.Unlock()

	b.original.ChangeName(n)
}
```

Enfin, on peut lier nos deux implémentations pour retourner un objet qui respecte l'interface **Book** :

```go
func CreateBook() Book {
  return &MutexBook{original: DefaultBook{}}
}
```

---

Le pattern **decorator** nous a aidé à séparer les responsabilités au travers de deux strucures.
