## Shopware data associations by examples

Database tables are often related to one another. For example, a blog post may have many comments or an order could be related to the user who placed it.

This blog post describes small examples of how to add associations to your entities. 

## One to One
A one-to-one association is a very basic type of database association. For example, a `User` table might be associated with one `Media` table. To define this association, we will place an `OneToOneAssociationField` on the `UserDefinition` and `MediaDefinition`. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651479650397/fcHd9hC1P.png align="left")

Let's have a look at the `defineFields` methods of both entity definitions:
```php
// class UserDefinition extends EntityDefinition
protected function defineFields(): FieldCollection
{
    return new FieldCollection([
        (new IdField('id', 'id'))->addFlags(new PrimaryKey(), new Required()),
        (new FkField('avatar_id', 'avatarId', MediaDefinition::class)),
        (new OneToOneAssociationField('avatarMedia', 'avatar_id', 'id', MediaDefinition::class)),
    ]);
}
```

```php
// class MediaDefinition extends EntityDefinition
protected function defineFields(): FieldCollection
{
    return new FieldCollection([
        (new IdField('id', 'id'))->addFlags(new PrimaryKey(), new Required()),
        (new OneToOneAssociationField('avatarUser', 'id', 'avatar_id', UserDefinition::class, false))->addFlags(new CascadeDelete()),
    ]);
}
```

## One to Many
A one-to-many association is used to define associations where a single table is a parent to one or more child tables. For example, a `Product` may have an infinite number of `Price`.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651479837383/-rcWkFmzT.png align="left")

Now, we will look at the example:

```php
// class ProductDefinition extends EntityDefinition
protected function defineFields(): FieldCollection
{
    return new FieldCollection([
        (new IdField('id', 'id'))->addFlags(new PrimaryKey(), new Required()),
        (new OneToManyAssociationField('prices', ProductPriceDefinition::class, 'product_id'))->addFlags(new CascadeDelete(), new Inherited()),
    ]);
}
```

```php
// class PriceDefinition extends EntityDefinition
protected function defineFields(): FieldCollection
{
    return new FieldCollection([
        (new IdField('id', 'id'))->addFlags(new PrimaryKey(), new Required()),
        (new FkField('product_id', 'productId', ProductDefinition::class))->addFlags(new Required()),
        (new ManyToOneAssociationField('product', 'product_id', ProductDefinition::class, 'id', false))->addFlags(new ReverseInherited('prices')),
    ]);
}
```

## Many to Many
Shopware also provides methods to make working with many-to-many associations more convenient. For example, let's imagine a `Product` can have many `Category` and a `Category` can have many `Product`.

This association require the third entity to be available. It will be called `ProductCategoryDefinition` and is responsible for connecting both definitions. It also needs its own database table `product_category`.

![Screen Shot 2022-05-02 at 15.36.06.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651480570349/LCAxmTFfM.png align="left")

```php
// class ProductDefinition extends EntityDefinition
protected function defineFields(): FieldCollection
{
    return new FieldCollection([
        (new IdField('id', 'id'))->addFlags(new PrimaryKey(), new Required()),
        (new VersionField())->addFlags(new ApiAware()),
        (new ManyToManyAssociationField('categories', CategoryDefinition::class, ProductCategoryDefinition::class, 'product_id', 'category_id'))->addFlags(new ApiAware(), new CascadeDelete(), new Inherited(), new SearchRanking(SearchRanking::ASSOCIATION_SEARCH_RANKING)),
    ]);
}
```

```php
// class CategoryDefinition extends EntityDefinition
protected function defineFields(): FieldCollection
{
    return new FieldCollection([
        (new IdField('id', 'id'))->addFlags(new PrimaryKey(), new Required()),
        (new VersionField())->addFlags(new ApiAware()),
        (new ManyToManyAssociationField('products', ProductDefinition::class, ProductCategoryDefinition::class, 'category_id', 'product_id'))->addFlags(new CascadeDelete(), new ReverseInherited('categories')),
    ]);
}
```

```php
// class ProductCategoryDefinition extends MappingEntityDefinition
protected function defineFields(): FieldCollection
{
    return new FieldCollection([
        (new FkField('product_id', 'productId', ProductDefinition::class))->addFlags(new PrimaryKey(), new Required()),
        (new ReferenceVersionField(ProductDefinition::class))->addFlags(new PrimaryKey(), new Required()),

        (new FkField('category_id', 'categoryId', CategoryDefinition::class))->addFlags(new PrimaryKey(), new Required()),
        (new ReferenceVersionField(CategoryDefinition::class))->addFlags(new PrimaryKey(), new Required()),

        (new ManyToOneAssociationField('product', 'product_id', ProductDefinition::class, 'id', false)),
        (new ManyToOneAssociationField('category', 'category_id', CategoryDefinition::class, 'id', false)),
    ]);
}
```

---

*I am really happy to receive your feedback on this article. Thanks for your precious time reading this.*

**Reference:** https://developer.shopware.com/docs/guides/plugins/plugins/framework/data-handling/add-data-associations