# Best Practices for Business Rules

## 🎯 Fundamental Principles

### 1. **Single Responsibility Principle**
Each Business Rule should have a clear, single responsibility.

❌ **Avoid:**
```magik
_method eo_pole.br_do_everything()
    ## Does validation, calculation, updates other objects...
    _self.validate_height()
    _self.calculate_load()
    _self.update_connected_equipment()
    _self.send_notification()
_endmethod
```

✅ **Prefer:**
```magik
_method eo_pole.br_validate_height()
    ## Validates only height
_endmethod

_method eo_pole.br_post_update_calculate_load()
    ## Calculates only load
_endmethod
```

### 2. **Fail Fast**
Validate and fail as early as possible.

✅ **Good:**
```magik
_method eo_pole.br_validate()
    # Quick checks first
    _if _self.asset_id _is _unset
    _then
        condition.raise(:user_error, :string, "Asset ID is mandatory")
    _endif

    # More complex checks after
    _if _self.has_invalid_geometry?()
    _then
        condition.raise(:user_error, :string, "Invalid geometry")
    _endif
_endmethod
```

### 3. **Avoid Side Effects in PRE Rules**
PRE rules should only validate, not modify data.

❌ **Avoid:**
```magik
_method eo_pole.br_pre_insert()
    _self.calculated_field << _self.do_complex_calculation()
_endmethod
```

✅ **Use POST or derived field:**
```magik
_method eo_pole.br_post_insert()
    _self.calculated_field << _self.do_complex_calculation()
_endmethod
```

## 🔍 Performance

### 1. **Avoid Heavy Queries**

❌ **Slow:**
```magik
_method eo_conductor.br_check_all_conductors()
    _local all_conductors << eo_conductor_inst.all_elements()
    _for cond _over all_conductors.fast_elements()
    _loop
        # validation
    _endloop
_endmethod
```

✅ **Fast:**
```magik
_method eo_conductor.br_check_nearby_conductors()
    _local bounds << _self.location.bounds.buffered_by(100)
    _local nearby << _self.source_view.world.geometry_at_bounds(bounds, :eo_conductor_inst)
    _for cond _over nearby.fast_elements()
    _loop
        # validation only for nearby
    _endloop
_endmethod
```

### 2. **Cache Expensive Results**

```magik
_method eo_power_xfrmr.br_validate_capacity()
    # Cache expensive calculation result
    _if _self.cached_capacity _is _unset
    _then
        _self.cached_capacity << _self.calculate_total_capacity()
    _endif

    _return _self.cached_capacity
_endmethod
```

### 3. **Use Indexes**
Make sure fields used in predicates have indexes.

```magik
# Good: indexed field
_local pred << predicate.eq(:asset_id, meter_number)

# Bad: non-indexed field
_local pred << predicate.eq(:custom_field, value)
```

## 🛡️ Error Handling

### 1. **Always Protect Critical Code**

```magik
_method eo_pole.br_complex_validation()
    _protect
        _local result << _self.do_risky_operation()
        _if result.failed?
        _then
            condition.raise(:user_error, :string, "Operation failed")
        _endif
    _protection
        _handling error _with _proc(cond)
            condition.raise(:user_error, :string,
                write_string("Validation error: ", cond.report_contents_string))
        _endproc
    _endprotect
_endmethod
```

### 2. **Clear and Actionable Messages**

❌ **Vague:**
```magik
condition.raise(:user_error, :string, "Error")
```

✅ **Specific:**
```magik
condition.raise(:user_error, :string,
    write_string("Invalid 'pole_height' field: ", value, ". ",
                 "Must be between 6 and 20 meters"))
```

### 3. **Use Message Handlers**

```magik
_method eo_pole.br_validate()
    _if _self.pole_height < 6.0
    _then
        condition.raise(:user_error, :string,
            _self.message(:pole_height_too_low, _self.pole_height))
    _endif
_endmethod

# In message file:
# :pole_height_too_low	Pole height too low: %1. Minimum: 6 meters
```

## 🧪 Testability

### 1. **Methods Should Be Testable**

```magik
_method eo_pole.br_validate_height()
    ## Public method, testable in isolation
    _return _self.private_validate_height_range()
_endmethod

_private _method eo_pole.private_validate_height_range()
    ## Isolated logic
    _if _self.pole_height < 6.0
    _then
        _return _false
    _endif
    _return _true
_endmethod
```

### 2. **Unit Tests**

```magik
_method test_pole.test_validate_height()
    ## Tests height validation

    _local pole << eo_pole.new()
    pole.pole_height << 5.0

    _self.assert_raises(
        :user_error,
        _proc() pole.br_validate_height() _endproc)
_endmethod
```

## 📊 Monitoring and Debug

### 1. **Appropriate Logging**

```magik
_method eo_pole.br_complex_operation()
    _if debug_mode?
    _then
        write("BR: Starting validation for pole ", _self.asset_id)
    _endif

    # validation

    _if debug_mode?
    _then
        write("BR: Validation completed successfully")
    _endif
_endmethod
```

### 2. **Timing Slow Operations**

```magik
_method eo_conductor.br_expensive_validation()
    _local start_time << system.elapsed_seconds()

    _protect
        # expensive operation
    _protection
        _local elapsed << system.elapsed_seconds() - start_time
        _if elapsed > 1.0
        _then
            write("WARNING: BR took ", elapsed, " seconds")
        _endif
    _endprotect
_endmethod
```

## 📋 Code Review Checklist

Before submitting a Business Rule, check:

- [ ] Single, clear responsibility
- [ ] Descriptive error messages
- [ ] Proper error handling
- [ ] Acceptable performance (< 1 second)
- [ ] Testable in isolation
- [ ] Adequate documentation
- [ ] Doesn't modify data in PRE rules
- [ ] Uses efficient predicates
- [ ] Avoids unnecessary queries
- [ ] Appropriate logging in complex operations

## 🚫 Common Anti-Patterns

### 1. **Infinite Loop**
❌ Modifying the same object inside its BR
```magik
_method eo_pole.br_post_update()
    _self.field << new_value  # Triggers the same BR again!
_endmethod
```

### 2. **BR Acting as Plugin**
❌ Interface logic in BR
```magik
_method eo_pole.br_show_dialog()
    _local dialog << _self.build_dialog()
    dialog.activate()
_endmethod
```

### 3. **Late Validation**
❌ Validating in POST when it should be PRE
```magik
_method eo_pole.br_post_insert()
    _if _self.asset_id _is _unset
    _then
        condition.raise(:user_error, ...)  # Too late!
    _endif
_endmethod
```

## 📚 Additional Resources

- [Business Rules Structure](./README.md)
- [Practical Examples](./Exemple/)
- [Mandatory Field Validation](./Exemple/Mandatory_Field_Validation.md)
- [PRE Insert Validation](./Exemple/PRE_Insert_Validation.md)
- [POST Update Action](./Exemple/POST_Update_Action.md)
- [Duplicate Validation](./Exemple/Duplicate_Validation.md)
