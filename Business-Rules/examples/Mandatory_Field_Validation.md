# Mandatory Field Validation

## Objective
Ensure that essential fields are filled before saving a record.

## Rule Type
**PRE INSERT / PRE UPDATE**

## When to Use
- Mandatory fields not defined in the data model
- Conditional validations (field A mandatory if field B = X)
- Specific business validations

## Complete Example

### 1. Create the validation method

```magik
_pragma(classify_level=basic, topic={business_rule})
_method eo_pole.br_validate_mandatory_fields()
    ## Validates if mandatory fields are filled
    ## Raises :user_error if any mandatory field is empty

    _local mandatory_fields << property_list.new_with(
        :asset_id, "Asset Number",
        :pole_height, "Pole Height",
        :pole_class, "Pole Class"
    )

    _for field_name, field_label _over mandatory_fields.fast_keys_and_elements()
    _loop
        _local value << _self.perform(field_name)

        _if value _is _unset _orif
            value = "" _orif
            (value.is_kind_of?(float) _andif value = 0.0)
        _then
            _local msg << write_string("Field '", field_label, "' is mandatory")
            condition.raise(:user_error, :string, msg)
        _endif
    _endloop

_endmethod
```

### 2. Rule installation script

```magik
_block
    ## Script to create Business Rule for mandatory field validation

    _local rule_name << :br_eo_pole_validate_mandatory
    _local table_name << "eo_pole"
    _local method_name << "br_validate_mandatory_fields()"

    # Remove existing rule if any
    _local existing << sw_gis!business_rule.get_by_name(rule_name)
    _if existing _isnt _unset
    _then
        existing.delete()
        write("Existing rule removed")
    _endif

    # Create new PRE INSERT rule
    _local br << sw_gis!business_rule.new_with(
        rule_name,
        table_name,
        "pre",
        "insert",
        method_name,
        _true  # active?
    )

    write("Business Rule created: ", br.rule_name)

    # Create PRE UPDATE rule as well
    _local br2 << sw_gis!business_rule.new_with(
        write_string(rule_name, "_update").as_symbol(),
        table_name,
        "pre",
        "update",
        method_name,
        _true
    )

    write("UPDATE Business Rule created: ", br2.rule_name)

    # Commit
    gis_program_manager.databases[:gis].commit()
    write("Commit successful!")

_endblock
```

### 3. How to test

```magik
# Manual test
_block
    _local pole << eo_pole.new()
    pole.asset_id << "POLE123"
    # Don't fill pole_height on purpose

    _protect
        pole.validate()  # Should raise error
        write("ERROR: Validation should have failed!")
    _protection
        _handling user_error _with _proc(cond)
            write("OK: Error caught - ", cond.report_contents_string)
        _endproc
    _endprotect
_endblock
```

## Variations

### Conditional Validation

```magik
_method eo_power_xfrmr.br_validate_conditional_fields()
    ## If transformer is three-phase, number of phases must be 3

    _if _self.phase_code = "ABC" _andif
        _self.number_of_phases <> 3
    _then
        condition.raise(:user_error, :string,
            "Three-phase transformer must have 3 phases")
    _endif

_endmethod
```

### Range Validation

```magik
_method eo_pole.br_validate_pole_height()
    ## Pole height must be between 6 and 20 meters

    _local height << _self.pole_height

    _if height _is _unset _then _return _endif

    _if height < 6.0 _orif height > 20.0
    _then
        condition.raise(:user_error, :string,
            "Pole height must be between 6 and 20 meters")
    _endif

_endmethod
```

## Tips

1. **Group related validations** in a single method
2. **Use clear messages** indicating which field and why it failed
3. **Consider internationalization** using message handlers
4. **Handle special cases** like system default values

## Next Steps

- [PRE Insert Validation](../02-pre-insert-validation/)
- [Relationship Validation](../05-relationship-validation/)
