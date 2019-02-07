---
layout: post
title:  "Migrating to Version 2 of the AWS Java SDK for DynamoDB"
author: "Rohan Nagar"
---

If you write in Java and use [Amazon Web Services (AWS)](https://aws.amazon.com/), you have most likely worked with the AWS SDK
for Java. For many years, the AWS Java SDK has been version `1.11.x`. However, recently the team at AWS has
[released version 2](https://aws.amazon.com/blogs/developer/aws-sdk-for-java-2-x-released/) of the AWS Java SDK. This blog post will
detail the changes I had to make in my application, [Thunder](https://github.com/RohanNagar/thunder), in order to migrate to the latest
and greatest version of the AWS SDK in order to interact with DynamoDB.

# Update Maven Dependencies

First, you will need to update your Maven dependencies. It is actually possible to
[use both version 2 and version 1.11.x side-by-side](https://docs.aws.amazon.com/sdk-for-java/v2/migration-guide/side-by-side.html)
if you wish to do so. For now, let's assume we want to fully migrate to version 2.

In your parent `pom.xml`, include the AWS SDK BOM:

{% highlight xml  linenos %}
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>software.amazon.awssdk</groupId>
      <artifactId>bom</artifactId>
      <version>${aws.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
{% endhighlight %}

Don't forget to include a property for the version.

{% highlight xml  linenos %}
<properties>
  <aws.version>2.3.9</aws.version>
</properties>
{% endhighlight %}

Now, in your `pom.xml` for the Maven module that you communicate to AWS in, add the artifacts that you will need.
For example, if you need to communicate with DynamoDB and SES, you will add the following:

{% highlight xml  linenos %}
<dependencies>
  <dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>dynamodb</artifactId>
  </dependency>
  <dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>ses</artifactId>
  </dependency>
</dependencies>
{% endhighlight %}

# Update Your DynamoDB Code

Most of the differences between `1.11.x` and `2.x` can be found [here](https://docs.aws.amazon.com/sdk-for-java/v2/migration-guide/whats-different.html).
In this post, we are focused on updating the code that communicates with DynamoDB. Let's start with how to create the client object.

Previously, you would create the DynamoDB client like so:

{% highlight java  linenos %}
AmazonDynamoDB client = AmazonDynamoDBClientBuilder.standard()
    .withEndpointConfiguration(new AwsClientBuilder.EndpointConfiguration(endpoint, region))
    .build();

DynamoDB dynamoDbClient = new DynamoDB(client);
{% endhighlight %}

And then you would actually operate on a `Table` object, so you would have to get the table as well with `dynamoDbClient.getTable(tableName)`.

In version 2, client creation is much simpler and you directly interact with the `DynamoDbClient` instead of the `Table` object:

{% highlight java  linenos %}
DynamoDbClient dynamoDbClient = DynamoDbClient.builder()
    .region(Region.of(region))
    .endpointOverride(URI.create(endpoint))
    .build();
{% endhighlight %}

That's it! Now let's use the client to make some requests.

## Create Item

Table items were previously represented by the `Item` class which would be constructed like so:

{% highlight java  linenos %}
Item item = new Item()
    .withPrimaryKey("email", user.getEmail().getAddress())
    .withLong("creation_time", now)
    .withJSON("document", toJson(mapper, user));
{% endhighlight %}

In version 2, table items are represented by a `Map<String, AttributeValue>` object. This can allow for a little more simplicity since
we are used to how a `Map` works in Java.

{% highlight java  linenos %}
Map<String, AttributeValue> item = new HashMap<>();

item.put("email", AttributeValue.builder().s(user.getEmail().getAddress()).build());
item.put("creation_time", AttributeValue.builder().n(String.valueOf(now)).build());
item.put("document", AttributeValue.builder().s(toJson(mapper, user)).build());
{% endhighlight %}

Note that there is no longer a specific method for adding a JSON attribute. Instead, simply put another `Map` entry with a `String` `AttributeValue`.

Now, to create the item in the table we need to use a `PutItemRequest` instead of directly calling a method that takes in an `Item` object.
Most of the new APIs in version 2 use the Builder pattern:

{% highlight java  linenos %}
PutItemRequest putItemRequest = PutItemRequest.builder()
    .tableName(tableName)
    .item(item)
    .expected(Collections.singletonMap("email",
        ExpectedAttributeValue.builder().exists(false).build()))
    .build();
{% endhighlight %}

Notice a few things here. First, we have to specify the `tableName`. This is because we are no longer interacting with a `Table` object, we are
interacting with the entire `DynamoDbClient`. Second, if we want to add any [conditions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithItems.html#WorkingWithItems.ConditionalUpdate)
to our request, we need to use the `.expected()` method when building the request. This method takes a `Map<String, ExpectedAttributeValue>` of
expected attributes. In the example above, we are specifying a condition that the value of the `email` attribute should **not** already exist in
the table.

Finally, we can make the request:

{% highlight java  linenos %}
dynamoDbClient.putItem(putItemRequest);
{% endhighlight %}

Make sure you update the Exceptions that you catch when making this request.
[Here is a table of exception name changes](https://docs.aws.amazon.com/sdk-for-java/v2/migration-guide/exception-changes.html)
for your reference.

## Get Item

To get an item, we again need to use a request object called `GetItemRequest`. Provide the `tableName` as well as the primary `key` of the
item that you want to look up. For example, if your primary key is `email`, then you will do the following:

{% highlight java  linenos %}
GetItemRequest request = GetItemRequest.builder()
    .tableName(tableName)
    .key(Collections.singletonMap("email", AttributeValue.builder().s(email).build()))
    .build();

GetItemResponse response = dynamoDbClient.getItem(request);
{% endhighlight %}

To read the item from the response, simply call `response.item()`.

## Update Item

Updating an item is the same process as creating it for the first time. You will again use `PutItemRequest`. DynamoDB will take care of
updating the old item with the same key and replacing it with the new item.

## Delete Item

Finally, let's see how to delete an item. For this, you will use the `DeleteItemRequest`.

{% highlight java  linenos %}
DeleteItemRequest deleteItemRequest = DeleteItemRequest.builder()
    .tableName(tableName)
    .key(Collections.singletonMap("email", AttributeValue.builder().s(email).build()))
    .expected(Collections.singletonMap("email",
        ExpectedAttributeValue.builder()
            .value(AttributeValue.builder().s(email).build())
            .exists(true)
            .build()))
    .build();

dynamoDbClient.deleteItem(deleteItemRequest);
{% endhighlight %}

Notice again that we define the table name, the primary key of the item to delete, and any conditions that we expect in order for Dynamo to
execute the delete.

# Conclusion

In conslusion, updating your Java application that communicates with DynamoDB to use version 2 of the AWS SDK can be done without too much trouble.
The main things to watch out for are:

1. Use the new way of constructing the `DynamoDbClient`.
2. Use `Map<String, AttributeValue>` instead of `Item` to represent an item.
3. Build your requests using the builder pattern and the associated SDK `Request` objects. Include the table name and any conditional write expectations
in the requests.
4. Update the exceptions you catch to use the [new names](https://docs.aws.amazon.com/sdk-for-java/v2/migration-guide/exception-changes.html).

Feel free to leave any comments or questions down below and I'd be happy to help!
