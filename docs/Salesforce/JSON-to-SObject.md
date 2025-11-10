# **JSON-to-SObject Upsert Parser – Documentation**

## ✅ **Purpose**

This Apex class processes JSON payloads sent from Salesforce Flow.
It converts JSON data into SObjects, maps fields, and performs an **upsert** operation using a matching external reference field.

This is useful for integrations where a Flow receives JSON from an external system (e.g., MuleSoft, API callout, middleware) and needs to automatically update Salesforce records.

---

## ✅ **Class Overview**

```apex
public class <ClassName> { ... }
```

| Feature                                   | Description |
| ----------------------------------------- | ----------- |
| Accepts JSON input from Flow              | ✅           |
| Deserializes JSON into typed Apex objects | ✅           |
| Maps data into SObject fields             | ✅           |
| Performs upsert                           | ✅           |
| Returns a status message to Flow          | ✅           |

---

## ✅ **Inner Classes**

### **1. FlowInput**

Represents the incoming request from Flow.

```apex
public class FlowInput {
    @InvocableVariable(required=true)
    public String payload; 
}
```

| Field     | Type   | Description                              |
| --------- | ------ | ---------------------------------------- |
| `payload` | String | JSON string containing a list of records |

---

### **2. DataPayload**

Represents a single record from the JSON input.
Fields should mirror JSON keys and map to Salesforce fields.

```apex
public class DataPayload {
    public String external_ref;
    public String date_value;
    public String description;
    public String amount;
    public String last_updated;
    public String address;
    public String status;
    public String total;
    public String reference_code;
}
```

> *Note: Field names may be adjusted to match the expected JSON structure.*

---

## ✅ **Invocable Method**

### `updateData(List<FlowInput> requests)`

```apex
@InvocableMethod(label='Update Records from JSON')
public static List<String> updateData(List<FlowInput> requests)
```

| Parameter  | Description                                  |
| ---------- | -------------------------------------------- |
| `requests` | List of Flow inputs containing JSON payloads |

| Returns      | Description                                          |
| ------------ | ---------------------------------------------------- |
| List<String> | Contains a success or error message returned to Flow |

---

## ✅ **Processing Logic**

| Step | Description                                             |
| ---- | ------------------------------------------------------- |
| 1    | Validate incoming Flow request                          |
| 2    | Deserialize JSON into a list of `DataPayload` objects   |
| 3    | Convert each payload into a Salesforce SObject instance |
| 4    | Parse date and number strings where required            |
| 5    | Perform **upsert** using a reference field              |
| 6    | Return a result message to Flow                         |

---

## ✅ **Field Mapping Logic (example)**

| JSON Field     | Salesforce Field  | Processing              |
| -------------- | ----------------- | ----------------------- |
| `external_ref` | `External_Ref__c` | Used for upsert         |
| `date_value`   | `Date_Value__c`   | Parsed into a `Date`    |
| `amount`       | `Amount__c`       | Parsed into a `Decimal` |
| `description`  | `Description__c`  | Direct mapping          |
| `status`       | `Status__c`       | Direct mapping          |

> Mapping can be expanded or modified based on business needs.

---

## ✅ **Error Handling**

| Condition                | Result                                  |
| ------------------------ | --------------------------------------- |
| No input received        | Returns `No Payload Received`           |
| Invalid JSON             | Returns `Invalid JSON payload: <error>` |
| JSON contains no records | Returns `No Payload Data Found`         |
| Upsert operation fails   | Returns `Error: <exception>`            |

All errors are returned to Flow for transparency and debugging.

---

## ✅ **Example JSON Input**

```json
[
  {
    "external_ref": "ABC123",
    "date_value": "01/07/2024",
    "description": "Sample update",
    "amount": "£100.00",
    "last_updated": "30/06/2024",
    "status": "Active"
  }
]
```

---

# ✅ **Generic SObject JSON Upsert Parser**

### **Apex Class**

```apex
public class GenericJSONUpsertParser {

    public class FlowInput {
        @InvocableVariable(required=true)
        public String payload; // JSON string
        @InvocableVariable(required=true)
        public String objectApiName; // Example: 'Account', 'Custom_Object__c'
        @InvocableVariable(required=true)
        public String externalIdField; // Example: 'External_Id__c'
    }

    @InvocableMethod(label='Upsert SObjects from JSON')
    public static List<String> upsertRecords(List<FlowInput> requests) {
        if (requests == null || requests.isEmpty())
            return new List<String>{'No input provided'};

        FlowInput input = requests[0];
        if (String.isBlank(input.payload))
            return new List<String>{'No JSON payload provided'};

        try {
            // Convert JSON to list of generic Maps
            List<Object> rawList = (List<Object>) JSON.deserializeUntyped(input.payload);

            List<SObject> recordsToUpsert = new List<SObject>();

            // Build SObject type dynamically
            SObjectType sObjType = Schema.getGlobalDescribe().get(input.objectApiName);
            if (sObjType == null)
                return new List<String>{'Invalid object API name: ' + input.objectApiName};

            for (Object item : rawList) {
                Map<String, Object> fieldMap = (Map<String, Object>) item;

                SObject sObj = sObjType.newSObject();

                // Assign fields dynamically
                for (String fieldName : fieldMap.keySet()) {
                    Object value = fieldMap.get(fieldName);
                    if (value != null) {
                        try {
                            sObj.put(fieldName, value);
                        } catch (Exception e) {
                            return new List<String>{
                                'Field assignment error on ' + fieldName + ': ' + e.getMessage()
                            };
                        }
                    }
                }

                recordsToUpsert.add(sObj);
            }

            // Perform upsert using provided external ID field
            Database.UpsertResult[] results = Database.upsert(recordsToUpsert, input.externalIdField, false);

            return new List<String>{'Processed ' + results.size() + ' record(s)'};

        } catch (Exception e) {
            return new List<String>{'JSON processing error: ' + e.getMessage()};
        }
    }
}
```

---

# ✅ What This Class Does

| Feature                                                 | Supported |
| ------------------------------------------------------- | --------- |
| Works with *any* standard or custom object              | ✅         |
| Uses Flow inputs to specify object name and external ID | ✅         |
| No hard-coded fields or object names                    | ✅         |
| Accepts a JSON array of multiple records                | ✅         |
| Returns a status message to Flow                        | ✅         |

---

# ✅ Example Flow Input

**payload**

```json
[
  {
    "External_Id__c": "123",
    "Name": "Test Record",
    "Amount__c": 55.25,
    "Status__c": "Active"
  },
  {
    "External_Id__c": "124",
    "Name": "Second Record",
    "Amount__c": 99.99,
    "Status__c": "Pending"
  }
]
```

**objectApiName**

```
Custom_Object__c
```

**externalIdField**

```
External_Id__c
```

✅ This will upsert two `Custom_Object__c` records using the `External_Id__c` field.

---

# ✅ Error Handling

| Condition               | Returned Message                                 |
| ----------------------- | ------------------------------------------------ |
| Missing input           | `"No input provided"`                            |
| Missing JSON            | `"No JSON payload provided"`                     |
| Invalid Object API name | `"Invalid object API name: X"`                   |
| Field assignment fails  | `"Field assignment error on <field>: <message>"` |
| JSON failure            | `"JSON processing error: <message>"`             |

---



