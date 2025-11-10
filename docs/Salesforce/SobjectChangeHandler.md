# **Generic SObject Change Handler – Documentation**

## ✅ **Purpose**

This Apex class tracks field changes on SObjects and automatically publishes integration events whenever configured fields change.

It is designed to:

* Be triggered from **Record-Triggered Flows**, Apex triggers, or batch processes
* Dynamically determine which fields to track via **Custom Metadata** (`Object_Fields_Change__mdt`)
* Serialize changed fields into a JSON payload
* Publish events to **Platform Events** (`Outbound_Integration_Event__e`)

This generic approach supports any object without hard-coding field names or object names.

---

## ✅ **Class Overview**

```apex
public class GenericObjectChangeHandler { ... }
```

| Feature                          | Description |
| -------------------------------- | ----------- |
| Detects field changes            | ✅           |
| Configurable via Custom Metadata | ✅           |
| Creates JSON payloads            | ✅           |
| Publishes to Platform Events     | ✅           |
| Supports any SObject             | ✅           |

---

## ✅ **Key Components**

### **1. Change Tracking Configuration**

* **Custom Metadata Type**: `Object_Fields_Change__mdt`
* **Field**: `Field_List__c`
* **DeveloperName**: Set to the SObject API Name (e.g., `Account`, `Contact`, `Custom_Object__c`)
* **Purpose**: Defines which fields are monitored for changes.

---

### **2. Integration Event**

* **Platform Event**: `Outbound_Integration_Event__e`
* **Payload**: Contains changed field names and values, plus record identifiers.
* **Fields**:

  * `Object__c` → SObject API name
  * `Payload__c` → JSON of changed fields
  * `RecordId__c` → Salesforce Record Id
  * `Request_Type__c` → Contextual type of request
  * `OH_Ref_Number__c` → Optional reference field (genericized)

---

### **3. Public Methods**

#### `processChangedFields(List<SObject> oldRecords, List<SObject> newRecords)`

```apex
public static void processChangedFields(List<SObject> oldRecords, List<SObject> newRecords)
```

**Purpose**: Detects changes on configured fields and generates integration events.

**Parameters**:

| Name         | Type          | Description                            |
| ------------ | ------------- | -------------------------------------- |
| `oldRecords` | List<SObject> | Records before changes (can be `null`) |
| `newRecords` | List<SObject> | Records after changes                  |

**Logic**:

1. Determine which fields to check via `Object_Fields_Change__mdt`.
2. Compare old vs new records.
3. For each changed field, build a `Map<String, String>` with field name and new value.
4. Serialize the map into JSON.
5. Call `createIntegrationEvents()` to queue the event.
6. Publish all queued events via `publishEvent()`.

---

### **4. Private Methods**

#### `createIntegrationEvents(String payload, String objectType, String requestType)`

* Converts the JSON payload into a `Platform Event` instance.
* Queues the event in `eventsToPublish`.

#### `publishEvent()`

* Publishes all queued events to the EventBus.
* Clears the event list if not in a test context.

---

### **5. Generic Processing Logic**

| Step | Description                                                     |
| ---- | --------------------------------------------------------------- |
| 1    | Retrieve fields to track from Custom Metadata (`Field_List__c`) |
| 2    | For each SObject, compare old vs new values                     |
| 3    | If a field changed, add to `changedFieldMap`                    |
| 4    | Include optional reference fields if present on record          |
| 5    | Serialize map into JSON and call `createIntegrationEvents()`    |
| 6    | After processing all records, call `publishEvent()`             |

---

### **6. Example JSON Payload (Published Event)**

```json
{
  "Id": "001XXXXXXXXXXXX",
  "Name": "Updated Record Name",
  "Status__c": "Active",
  "Amount__c": 250.50
}
```

---

### **7. Error Handling**

| Condition                        | Action                       |
| -------------------------------- | ---------------------------- |
| No fields configured in metadata | No events published          |
| Invalid SObject                  | Throws a runtime exception   |
| Field does not exist on SObject  | Skips that field             |
| Event publish fails              | Exception thrown by EventBus |

---

### **8. Flow / Trigger Usage**

* Can be called from **Record-Triggered Flows**:

  * Pass old and new records as `SObject` lists.
  * The handler automatically detects changes.
* Can be reused for **any object** by creating the appropriate **Custom Metadata record**.
* Platform Events can then be subscribed to by downstream systems.

---

# ✅ **Generic Apex Version**

```apex
public class GenericObjectChangeHandler {
    public static List<Outbound_Integration_Event__e> eventsToPublish = new List<Outbound_Integration_Event__e>();
    public static List<String> fieldsToCheck = new List<String>();

    public static void processChangedFields(List<SObject> oldRecords, List<SObject> newRecords) {
        if (newRecords == null || newRecords.isEmpty()) return;

        fieldsToCheck.clear();
        String objectName = String.valueOf(newRecords[0].getSObjectType());
        
        Object_Fields_Change__mdt config = [
            SELECT Field_List__c 
            FROM Object_Fields_Change__mdt 
            WHERE DeveloperName = :objectName
            LIMIT 1
        ];

        if (config.Field_List__c != null) {
            for (String f : config.Field_List__c.split(',')) {
                if (!String.isBlank(f)) fieldsToCheck.add(f.trim());
            }
        }

        processRecords(oldRecords, newRecords, fieldsToCheck, objectName);
    }

    private static void processRecords(List<SObject> oldRecords, List<SObject> newRecords, List<String> fieldsToCheck, String objectName) {
        Schema.SObjectType sObjectType = newRecords[0].getSObjectType();
        Map<String, Schema.SObjectField> fieldMap = sObjectType.getDescribe().fields.getMap();

        for (Integer i = 0; i < newRecords.size(); i++) {
            SObject newRec = newRecords[i];
            SObject oldRec = (oldRecords != null && oldRecords.size() > i) ? oldRecords[i] : null;
            Map<String, String> changedFieldMap = new Map<String, String>();

            for (String fieldName : fieldsToCheck) {
                Object oldVal = (oldRec != null) ? oldRec.get(fieldName) : null;
                Object newVal = newRec.get(fieldName);

                Boolean isChanged = (oldRec == null) 
                    ? (newVal != null) 
                    : (oldVal == null && newVal != null) || (oldVal != null && !oldVal.equals(newVal));

                if (isChanged) changedFieldMap.put(fieldName, String.valueOf(newVal));
            }

            if (!changedFieldMap.isEmpty()) {
                changedFieldMap.put('Id', String.valueOf(newRec.get('Id')));
                String payload = JSON.serialize(changedFieldMap);
                createIntegrationEvents(payload, objectName, 'Update');
            }
        }

        publishEvent();
    }

    private static void createIntegrationEvents(String payload, String objectType, String requestType){
        Map<String, Object> payloadMap = (Map<String, Object>) JSON.deserializeUntyped(payload);
        String recordId = String.valueOf(payloadMap.get('Id'));

        eventsToPublish.add(new Outbound_Integration_Event__e(
            Object__c = objectType,
            Payload__c = payload,
            RecordId__c = recordId,
            Request_Type__c = requestType
        ));
    }

    private static void publishEvent(){
        if (!eventsToPublish.isEmpty()) {
            EventBus.publish(eventsToPublish);
            if (!Test.isRunningTest()) eventsToPublish.clear(); 
        }
    }
}
```

---

This generic version:

* Works with **any SObject**
* Reads **fields to track** dynamically from metadata
* Publishes **Platform Events** automatically
* No hard-coded object references

---

