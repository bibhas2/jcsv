# Zero Copy CSV File Parser in Java
Here we develop a CSV file parser that doesn't allocate any memory or copy data to parse a CSV file. This is done using the following techniques:

- Memory map the file
- Read the fields in a line as ``ByteBuffer`` slices
- Keep all text as ``ByteBuffer`` and never convert to Java's String

This implementation follows [RFC 4180](https://www.rfc-editor.org/rfc/rfc4180).

## Basic Example
Let's say that you have a CSV file that has the prices and number of items sold.

```
Item ID,Price,Orders
P001, 23.99, 12
K192, 11.95, 11
SK182, 33.45, 5
```

We can calculate the total revenue like this.

```java
@Test
public void parseDoubleTest() throws Exception {
    Parser p = new Parser();
    double[] total = {0.0};

    p.parse("test-data/products.csv", 10, record -> {
        if (record.lineIndex() == 0) {
            //Skip the header row
            return; 
        }

        double price = record.doubleField(1);
        int quantity = record.intField(2);

        total[0] += price * quantity;
    });

    assertEquals(586.58, total[0], 0.001);
}
```

## What is it Good for?
``ByteBuffer`` is very similar to ``std::string_view`` in C++. Their hidden power comes from the fact that they can be compared and sorted. All manners of benefit come from that. For example, they can be used as keys in a hash map.

I'm hoping that this CSV parser will be good at processing large CSV files in a memory constrainted environment. In general, this library will be a good fit where the entire content of the CSV needs to be loaded into memory. Example uses case will include:

- Sorting a CSV file.
- Searching repeatedly within the file based on a key. You can store the key field (which is a ``ByteBuffer``) in a hash map to speed up searching.

In the example below we lookup the revenue for a product using its ID.

```java
@Test
public void comparisonTest() throws Exception {
    Parser p = new Parser();
    var map = new HashMap<ByteBuffer, Double>();

    p.parse("test-data/products.csv", 10, record -> {
        if (record.lineIndex() == 0) {
            //Skip the header row
            return; 
        }

        ByteBuffer productId = record.field(0);
        double price = record.doubleField(1);
        int quantity = record.intField(2);

        //Store the revenue for a product
        map.put(productId, price * quantity);
    });

    //Lookup the revenue for product "K192".
    var key = ByteBuffer.wrap(
        "K192".getBytes(StandardCharsets.UTF_8));

    assertEquals(131.45, map.get(key), 0.001);
}
```