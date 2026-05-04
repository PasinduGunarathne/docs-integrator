---
title: Types
---

# Types

Type artifacts define the data structures used throughout your integration. They ensure type safety across services, event handlers, and transformations. Define types in dedicated `.bal` files and reuse them across all artifacts in your project.

The visual designer exposes five type kinds: **Record**, **Enum**, **Service Class**, **Union**, and **Array**. Each kind maps to a Ballerina type definition in source.

## Adding a type

1. Open the **WSO2 Integrator: BI** sidebar in VS Code.

   ![WSO2 Integrator sidebar showing the project structure with Types listed](/img/develop/integration-artifacts/supporting/types/step-1.png)

2. Click **+** next to **Types** in the sidebar (or click **+ Add Type** from the Types canvas).

3. In the **New Type** panel, choose **Create from scratch** or **Import** to bring in an existing definition.

   ![New Type creation form showing Kind and Name fields](/img/develop/integration-artifacts/supporting/types/step-2.png)

   | Field | Description |
   |---|---|
   | **Kind** | The type kind: **Record**, **Enum**, **Service Class**, **Union**, or **Array**. |
   | **Name** | A unique name for the type (for example, `OrderRequest`). |
   | **Members / Fields / Resource Methods** | The structural members for the selected kind. The label changes to match the kind. |
   | **Advanced Options** | Per-kind options such as **Allow Additional Fields**, **Is Readonly Type**, and **Accessible by Other Integrations**. |

4. Click **Save**. The type is added to your project and appears in the type diagram.

```ballerina
// types.bal

type OrderRequest record {|
    string customerId;
    LineItem[] items;
    Address shippingAddress;
    string? couponCode;
|};

type LineItem record {|
    string productId;
    string productName;
    int quantity;
    decimal unitPrice;
|};

type Address record {|
    string street;
    string city;
    string state;
    string zipCode;
    string country;
|};
```

## Type diagram

Click **View Type Diagram** (or the diagram icon) next to **Types** in the sidebar to open the visual type diagram. The canvas renders all types in your project as nodes, with arrows showing relationships between record types and their nested fields.

![Type diagram canvas showing the visual representation of types in the project](/img/develop/integration-artifacts/supporting/types/step-3.png)

Use the toolbar buttons at the bottom left to zoom in, zoom out, fit the diagram to the screen, or export it as an image.

Type relationships are expressed through field type references. A record that contains another record as a field creates an implicit association in the diagram:

```ballerina
type OrderRequest record {|
    string customerId;
    LineItem[] items;      // references LineItem
    Address shippingAddress; // references Address
|};
```

## Type kinds

### Record

A record defines a structured type with named fields.

Select **Record** from the **Kind** dropdown, then add one row per field using **+**.

For each field:

| Control | Description |
|---|---|
| **Name** | The field name. |
| **Type** | The field type (built-in, user-defined, or another record). |
| **Inline record** (`{}`) | Define the field's type as a nested record inline, instead of referencing an existing type. |
| **Optional** (`?`) | Mark the field optional. The field becomes `T?` and may be absent at runtime. |
| **Delete** (trash) | Remove the field. |

Expand a field row to set:

| Option | Description |
|---|---|
| **Default Value** | A literal expression assigned when the field is omitted. The field is then defaultable in source. |
| **Description** | Documentation for the field, emitted as a Ballerina doc comment. |
| **Readonly** | Mark this field immutable. The field becomes `readonly T`. |

**Advanced Options**:

| Option | Description |
|---|---|
| **Allow Additional Fields** | Generate an open record (`record { ... }`) that accepts unknown fields. Leave unchecked for a closed record (`record {\| ... \|}`). |
| **Is Readonly Type** | Wrap the whole type as `readonly & T` so every field is immutable. |
| **Accessible by Other Integrations** | Add the `public` qualifier so the type can be imported from other packages. |

```ballerina
# Represents an order placed by a customer.
public type OrderRequest record {|
    # Unique identifier of the customer.
    readonly string customerId;
    LineItem[] items;
    Address shippingAddress;
    # Optional discount coupon.
    string? couponCode;
    string currency = "USD";
|};
```

### Enum

An enum defines a fixed set of named string values.

Select **Enum** from the **Kind** dropdown, then add each member using **+**.

For each member:

| Control | Description |
|---|---|
| **Enum member name** | The member identifier (for example, `PENDING`). |
| **Constant Expression** | The string value bound to the member. Leave empty to use the member name as the value. |

```ballerina
enum OrderStatus {
    PENDING,
    CONFIRMED = "confirmed",
    SHIPPED = "shipped",
    DELIVERED = "delivered",
    CANCELLED
}
```

### Service class

A service class defines an object that exposes resource methods, used to model dynamically generated services such as GraphQL resolvers.

#### Create a service class

Select **Service Class** from the **Kind** dropdown, then add one row per resource method using **+**.

For each resource method:

| Control | Description |
|---|---|
| **Name** | The resource path/name (for example, `name1`). |
| **Type** | The return type of the resource method. |
| **+ Add Parameter** | Add an input parameter. Each parameter takes a **Parameter Name**, **Parameter Type**, and an optional **Default Value**. |
| **Delete** (trash) | Remove the resource method. |

#### Edit the service class

The editing experience for a service class differs from other kinds. Click **Edit** on the type to open the **Edit Type: Service Class** panel:

| Control | Description |
|---|---|
| **Object Name** | Rename the service class. |
| **Description** | Free-text documentation rendered as a Ballerina doc comment. |
| **Fields** | List of resource methods. Use **+** to add, the pencil icon to edit a field, or the trash icon to remove it. |
| **Implement** | Switch to the **Service Class Designer** to add class variables and write method bodies. |

#### Edit a field

Click the pencil icon on a field to open the **Edit Field** panel:

| Control | Description |
|---|---|
| **Field Name** | The resource path/name. |
| **Description** | Documentation for the field. |
| **Arguments** | Use **+ Add Argument** to define input parameters for the resource method. |
| **Field Type** | The return type (for example, `string\|error`). |

Expand **Advanced Configurations** to set:

| Option | Description |
|---|---|
| **Field Configuration** | Free-form JSON configuration such as field-level cache settings. |
| **Request Context** | Inject a context value used to pass meta-information among GraphQL resolvers. |
| **Field Metadata** | Inject the field metadata used to access information about the field in a GraphQL document. |

#### Service Class Designer

Click **Implement** to open the **Service Class Designer**, where you add class state and method implementations.

![Service Class Designer showing Class Variables and Methods sections](/img/develop/integration-artifacts/supporting/types/service-class-designer.png)

**Class Variables** — click **+ Variable** to open the **Add Variable** panel:

| Control | Description |
|---|---|
| **Variable Name** | The variable identifier. |
| **Variable Type** | The variable type. |
| **Default Value** | The initial value assigned at construction. |

**Methods** — click **+ Method** to open the **Add Method** panel. Choose the method kind from the dropdown — **resource**, **remote**, or **init** — then fill in:

| Control | Description |
|---|---|
| **Function Name** | The method name. Not shown for `init`. |
| **Description** | Documentation for the method. |
| **Parameters** | Use **+ Add Parameter** to add inputs (name, type, optional default value). |
| **Return Type** | The method return type. Not shown for `init`. |

Use the pencil and trash icons next to each existing method to edit or remove it.

```ballerina
# Represents a pet shelter exposed through GraphQL.
public service class Shelter {
    private final string name;
    private final string location;
    private int capacity;

    function init(string name, string location, int capacity) {
        self.name = name;
        self.location = location;
        self.capacity = capacity;
    }

    # Returns the shelter name.
    resource function get name() returns string|error {
        return self.name;
    }

    resource function get location() returns string {
        return self.location;
    }

    resource function get capacity() returns int {
        return self.capacity;
    }

    remote function updateCapacity(int newCapacity) returns int {
        self.capacity = newCapacity;
        return self.capacity;
    }
}
```

### Union

A union allows a value to be one of several member types.

Select **Union** from the **Kind** dropdown, then add each member type using **+**. Members can be built-in types or other named types.

**Advanced Options**:

| Option | Description |
|---|---|
| **Is Readonly Type** | Wrap the union as `readonly & T` so values are immutable. |
| **Accessible by Other Integrations** | Add the `public` qualifier so the type can be imported from other packages. |

```ballerina
public type PaymentMethod CreditCard|BankTransfer|DigitalWallet;

type CreditCard record {|
    string cardNumber;
    string expiryDate;
    string cvv;
|};

type BankTransfer record {|
    string bankName;
    string accountNumber;
    string routingNumber;
|};

type DigitalWallet record {|
    string provider;
    string walletId;
|};
```

### Array

An array defines an ordered list of values of a single member type.

Select **Array** from the **Kind** dropdown.

| Control | Description |
|---|---|
| **Type of the Array** | The member type (required). |
| **Size of the Array** | A fixed length. Leave empty for an unbounded array. |

**Advanced Options**:

| Option | Description |
|---|---|
| **Is Readonly Type** | Wrap the array as `readonly & T[]` so it cannot be mutated. |
| **Accessible by Other Integrations** | Add the `public` qualifier so the type can be imported from other packages. |

```ballerina
// Unbounded array
public type LineItems string[];

// Fixed-length array
type Coordinates float[3];
```

## Map types

The visual designer does not expose **Map** as a Kind. To model dynamic key/value data, define a record instead:

- For a known set of keys, use a closed record (`record {| ... |}`).
- For arbitrary string keys with a uniform value type, use an open record with **Allow Additional Fields** enabled.

If you are editing source directly, the equivalent Ballerina form is `map` — for example, `map<string>` for a string-valued map. Types declared this way appear as opaque references in the type diagram.

## Best practices

| Practice | Description |
|---|---|
| **Closed records** | Leave **Allow Additional Fields** unchecked unless you actually need extra keys. Closed records catch typos at compile time. |
| **Dedicated files** | Keep type definitions in separate `types.bal` files and import them where needed. |
| **Descriptive names** | Name types after their domain concept (for example, `OrderRequest`, not `Data`). |
| **Reuse across artifacts** | Define types once and reference them from services, event handlers, and functions. |
| **Prefer Readonly over reassignment** | Mark fields or whole types **Readonly** for immutable data, instead of relying on convention. |
| **Use records instead of maps in the designer** | The designer cannot edit `map` types. Model dynamic data as records (open records when keys are unknown). |
