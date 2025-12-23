# Business Rules in Smallworld GIS

## 📋 Table of Contents
- [What are Business Rules?](#what-are-business-rules)
- [When to use Business Rules?](#when-to-use-business-rules)
- [Basic Structure](#basic-structure)
- [Types of Business Rules](#types-of-business-rules)
- [Practical Examples](#practical-examples)
- [Best Practices](#best-practices)

## What are Business Rules?

Business Rules are Smallworld mechanisms that allow you to intercept database operations (insert, update, delete) and execute custom validations or actions.

**Key characteristics:**
- Execute automatically when certain conditions are met
- Can validate data before it's saved
- Can execute actions after database operations
- Configured through records in the `sw_gis!business_rule` table

## When to use Business Rules?

✅ **Use Business Rules for:**
- Validating mandatory data
- Validating consistency of related data
- Automating field calculations
- Preventing invalid operations
- Executing cascade actions
- Integrating with external systems

❌ **Avoid using for:**
- Complex interface logic (use plugins)
- Batch operations (use direct methods)
- Validations that already exist in the data model

## Basic Structure

A Business Rule has:

1. **Configuration Record** - Defines when the rule will execute
2. **Implementation Method** - Magik code that will be executed

### Anatomy of a BR

```
business_rule
├── table_name          # Target table (e.g., eo_pole)
├── rule_type          # Type: pre/post/check
├── operation          # Operation: insert/update/delete
├── method_name        # Method to be executed
└── active?            # Active/Inactive
```

## Types of Business Rules

### 1. **PRE Rules** (Before operation)
- Execute **before** insert/update/delete
- Can **prevent** the operation (by raising condition)
- Used for **validations**

### 2. **POST Rules** (After operation)
- Execute **after** insert/update/delete
- **Cannot** prevent the operation
- Used for **automatic actions**

### 3. **CHECK Rules** (Verification)
- Execute during field validations
- Real-time validations in the interface

## Practical Examples

Check detailed examples in the Examples folder:

- [Mandatory Field Validation](./Exemple/Mandatory_Field_Validation.md)
- [PRE Insert Validation](./Exemple/PRE_Insert_Validation.md)
- [POST Update Action](./Exemple/POST_Update_Action.md)
- [Duplicate Validation](./Exemple/Duplicate_Validation.md)

## Best Practices

### ✅ Do

1. **Always handle conditions**
   ```magik
   _protect
       # your code
   _protection
       _handling error _with procedure
       _endprotocol
   _endprotect
   ```

2. **Use descriptive messages**
   ```magik
   condition.raise(:user_error, :string, "Field X is mandatory")
   ```

3. **Check if field changed (in updates)**
   ```magik
   _if old_record.field_name ~= new_record.field_name
   _then
       # validation
   _endif
   ```

4. **Document your rules**
   ```magik
   ## Validates if field X is filled before insert
   ## Raises :user_error if field is empty
   _method eo_pole.validate_field_x()
   ```

### ❌ Avoid

1. ❌ Processing large datasets in PRE rules
2. ❌ Modifying the same object inside the rule (infinite loop!)
3. ❌ Raising conditions in POST rules
4. ❌ Complex database operations in synchronous rules

## Recommended File Structure

```
your_module/
├── source/
│   └── business_rules/
│       ├── br_validations.magik
│       └── br_actions.magik
└── scripts/
    └── install/
        └── create_business_rules.magik
```

## Debugging

### How to test your Business Rules

```magik
# 1. Temporarily disable
sw_gis!business_rule.get_by_name(:your_rule).active? << _false

# 2. Manually test the method
_local obj << eo_pole.new()
obj.your_br_method()

# 3. View all active rules for a table
sw_gis!business_rule.select(:table_name|=|"eo_pole").do_query()
```

## References

- [Official GE Smallworld Documentation](https://www.ge.com/digital/documentation)
- [Examples in this repository](./examples/)

---

**Next topics:**
- [Triggers vs Business Rules](./advanced/triggers-vs-br.md)
- [Performance in Business Rules](./advanced/performance.md)
- [Unit Tests for BR](./advanced/unit-tests.md)
