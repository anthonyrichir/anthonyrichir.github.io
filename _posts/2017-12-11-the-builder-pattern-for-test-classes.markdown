---
layout: post
title:  "The builder pattern for test classes"
date:   2017-12-11 10:24:10 +0100
categories: java pattern
---
I’ve always been interested in software testing, either unit tests, integration tests, ui tests, … Not because it’s the funniest part of software development but because I think it’s important to deliver quality and testing is the main way to achieve it.

Recently, I wrote a lot of integration tests for an API with quite complex object trees and to make effective tests, I need to insert effective data.

Our APIs are generated using Jhipster, the following code snippet shows how a Jhipster application creates and persists objects in integration tests.

A `createEntity` method is generated, being responsible for instantiating a class, eventually calling a `createEntity` from another test class to create an instance of a dependency.

```java
public class NodeResourceIntTest {

    @Autowired
    private EntityManager em;

    private Node node;

    public static Node createEntity(EntityManager em) {
        node = new Node()
            .lft(DEFAULT_LFT)
            .rgt(DEFAULT_RGT)
            .nodeOnline(DEFAULT_NODE_ONLINE)
            .lastChangeAt(DEFAULT_LAST_CHANGE_AT);
        // Add required entity
        Tree tree = TreeResourceIntTest.createEntity(em);
        em.persist(tree);
        em.flush();
        node.setTree(tree);
        return node;
    }

    @Before
    public void initTest() {
        node = createEntity(em);
    }
}
```

This example remains simple. What if I tell you that a Tree object is also linked to several children of type Nodes, themselves having a set of MetaGroup, each MetaGroup having a MetaType and a set of Meta, … and so on.

Some tests don’t need the full hierarchy but for some others, a full hierarchy is required for the test to be relevant.

A convenient way to create objects is the builder pattern. Creating a Node object could look like this.

```java
Node node = new NodeBuilder()
    .withLft(DEFAULT_LFT)
    .withRgt(DEFAULT_RGT)
    .withNodeOnline(DEFAULT_NODE_ONLINE)
    .withLastChangeAt(DEFAULT_LAST_CHANGE_AT)
    .build();
```

Good enough for a unit test, but I also do a lot of integration tests and therefore, I want my objects to be added to my persistence context and stored in my database. Therefore, I added a `buildAndPersist` method to my builder and to be able to persist it, I needed an EntityManager, which I pass to the builder via the constructor.

```java
Node node = new NodeBuilder(entityManager)
    .withLft(DEFAULT_LFT)
    .withRgt(DEFAULT_RGT)
    .withNodeOnline(DEFAULT_NODE_ONLINE)
    .withLastChangeAt(DEFAULT_LAST_CHANGE_AT)
    .buildAndPersist();
```

I ended up building an abstract class, so that all my builders would follow the same behavior.

```java
public abstract class AbstractPersistenceTestBuilder<T> {

    private final EntityManager em;

    public AbstractPersistenceTestBuilder(EntityManager em) {
        this.em = em;
    }

    public T buildAndPersist() {
        // Building the object via the implementation from the concrete class
        T target = build();
        // Persisting the created object
        em.persist(target);
        // Returning the instance linked to the persistence context
        return target;
    }

    public abstract T build();
}
```

From now on, building a complex object can be as simple as this code snippet, which in my opinion, also make the code much easier to read.

```java
Tree tree = new TreeTestBuilder(em)
      .withName("firstTree")
      .withCategory("category")
      .withCode(UUID.randomUUID().toString())
      .withLastChangeAt(now())
      .withMetaGroups(newHashSet(new MetaGroupTestBuilder(em)
            .withMetaType(new MetaTypeTestBuilder(em)
                  .withName("title")
                  .withProtectedType(TRUE)
                  .withValueType(MetaTypeValueType.TEXT)
                  .withLastChangeAt(now())
                  .buildAndPersist())
            .withMetas(newHashSet(new MetaTestBuilder(em)
                  .withMetaValue("my lovely title")
                  .withContextLanguage(ContextLanguage.EN)
                  .withLastChangeAt(now())
                  .buildAndPersist()))
            .buildAndPersist()))
      .withLabels(newHashSet(labelLevel))
      .buildAndPersist();
```

The source code is available on [my Github][github-test-builder].

In a next post, I hope to publish this very simple library to Maven Central as an exercise.

[github-test-builder]: https://github.com/anthonyrichir/test-builder