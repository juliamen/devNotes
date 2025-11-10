## Parse Date dd/MM/yyyy into Date Documentation

### `parseDate(String s)`

Parses a date string in **`dd/MM/yyyy`** format and converts it into a `Date` value.

#### **Method Signature**

```apex
private static Date parseDate(String s)
```

#### **Parameters**

| Name | Type     | Description                                                |
| ---- | -------- | ---------------------------------------------------------- |
| `s`  | `String` | The date string to convert. Expected format: `dd/MM/yyyy`. |

#### **Returns**

| Type   | Description                                                                          |
| ------ | ------------------------------------------------------------------------------------ |
| `Date` | A `Date` instance parsed from the string, or `null` if the input is `null` or empty. |

#### **Logic**

1. If the input string is `null` or empty (`''`), the method returns `null`.
2. The string is split by `'/'`, producing `[day, month, year]`.
3. `Date.newInstance(year, month, day)` creates the Date value.

#### **Example**

```apex
Date d = parseDate('25/12/2024');
// Returns 2024-12-25
```

---

### `parseDecimal(String s)`

Converts a formatted currency string into a numeric `Decimal`.

#### **Method Signature**

```apex
private static Decimal parseDecimal(String s)
```

#### **Parameters**

| Name | Type     | Description                                                                |
| ---- | -------- | -------------------------------------------------------------------------- |
| `s`  | `String` | A numeric string, optionally containing a currency symbol (`£`) or spaces. |

#### **Returns**

| Type      | Description                                         |
| --------- | --------------------------------------------------- |
| `Decimal` | Parsed decimal value, or `null` if input is `null`. |

#### **Logic**

1. If `s` is `null`, return `null`.
2. Remove `£` and spaces using `replace()`.
3. Convert to `Decimal` using `Decimal.valueOf()`.

#### **Example**

```apex
Decimal cost = parseDecimal(' £1,250.50 ');
// Returns 1250.50
```

---


```apex
/**
 * Parses a date string in dd/MM/yyyy format and returns a Date object.
 * @param s String date in dd/MM/yyyy
 * @return Date or null if input is null or empty
 */
private static Date parseDate(String s) {
    if (s == null || s == '') return null;
    List<String> parts = s.split('/');
    return Date.newInstance(
        Integer.valueOf(parts[2]),
        Integer.valueOf(parts[1]),
        Integer.valueOf(parts[0])
    );
}

/**
 * Converts a formatted currency/number string (e.g. "£1,000.50") to Decimal.
 * @param s String representing a decimal value
 * @return Decimal parsed value or null if input is null
 */
private static Decimal parseDecimal(String s) {
    if (s == null) return null;
    s = s.replace('£','').replace(' ','');
    return Decimal.valueOf(s);
}
```

---

