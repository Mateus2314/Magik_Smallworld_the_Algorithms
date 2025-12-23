# POST Update Action

## Objective
Execute actions automatically after a record is updated in the database.

## Rule Type
**POST UPDATE**

## When to Use
- Recalculate dependent data
- Update related objects
- Trigger cascade updates
- Log changes for audit
- Notify other systems

## Complete Examples

### Example 1: Recalculate Transformer Bank

```magik
_pragma(classify_level=basic, topic={business_rule})
_method eo_power_xfrmr.br_post_update_recalculate_bank()
    ## Business Rule POST UPDATE
    ## When transformer is updated, recalculate bank data
    ##
    ## Actions:
    ## - Update bank total capacity
    ## - Update bank status
    ## - Log change

    _local xfrmr_bank << _self.follows(:eo_power_xfrmr_bank)

    _if xfrmr_bank _is _unset
    _then
        _return  # Transformer not in a bank
    _endif

    # Recalculate bank total capacity
    _local total_capacity << 0.0
    _local all_xfrmrs << xfrmr_bank.follows(:eo_power_xfrmr)

    _for xfrmr _over all_xfrmrs.fast_elements()
    _loop
        _local capacity << xfrmr.rated_capacity
        _if capacity _isnt _unset
        _then
            total_capacity +<< capacity
        _endif
    _endloop

    # Update bank
    xfrmr_bank.total_capacity << total_capacity

    write("✓ Bank updated. Total capacity: ", total_capacity, " kVA")

_endmethod
$
```

### Example 2: Audit Log

```magik
_pragma(classify_level=basic, topic={business_rule})
_method eo_pole.br_post_update_audit_log()
    ## Logs important changes to audit log
    ## Monitored fields: pole_height, pole_class, owner

    _local old_rec << _self.old_version
    _if old_rec _is _unset _then _return _endif

    _local monitored_fields << {:pole_height, :pole_class, :owner}
    _local changes << rope.new()

    _for field _over monitored_fields.fast_elements()
    _loop
        _local old_value << old_rec.perform(field)
        _local new_value << _self.perform(field)

        _if old_value ~= new_value
        _then
            changes.add_last({field, old_value, new_value})
        _endif
    _endloop

    _if changes.empty?
    _then
        _return  # No changes in monitored fields
    _endif

    # Here you would create an audit record
    # For example, in an audit_log table
    write("=== AUDIT: POLE ", _self.asset_id, " ===")
    _for change _over changes.fast_elements()
    _loop
        _local (field, old_val, new_val) << (_scatter change)
        write("  ", field, ": ", old_val, " → ", new_val)
    _endloop

_endmethod
$
```

### Example 3: Notify Downstream Equipment

```magik
_pragma(classify_level=basic, topic={business_rule})
_method eo_conductor_inst.br_post_update_notify_downstream()
    ## When conductor changes specification, notify downstream equipment
    ## to recalculate their settings

    _local old_rec << _self.old_version
    _if old_rec _is _unset _then _return _endif

    # Check if spec changed
    _if old_rec.conductor_spec = _self.conductor_spec
    _then
        _return  # Spec didn't change
    _endif

    write("Conductor spec changed. Recalculating downstream equipment...")

    # Find all connected equipment
    _local connected_equipment << _self.follows(:connectivity, eo_equipment)

    _for equip _over connected_equipment.fast_elements()
    _loop
        # Trigger recalculation on equipment
        _if equip.responds_to?(:recalculate_settings|()|)
        _then
            equip.recalculate_settings()
            write("  ✓ Recalculated: ", equip.asset_id)
        _endif
    _endloop

_endmethod
$
```

### Example 4: Update Calculated Fields

```magik
_pragma(classify_level=basic, topic={business_rule})
_method eo_service_point.br_post_update_calculate_load()
    ## Recalculates total load when customer load changes

    _local old_rec << _self.old_version
    _if old_rec _is _unset _then _return _endif

    # Check if load fields changed
    _local load_changed? << _false
    _for field _over {:demand_kw, :power_factor, :phase_code}.fast_elements()
    _loop
        _if old_rec.perform(field) ~= _self.perform(field)
        _then
            load_changed? << _true
            _leave
        _endif
    _endloop

    _if _not load_changed?
    _then
        _return
    _endif

    # Recalculate total apparent power
    _local demand_kw << _self.demand_kw
    _local pf << _self.power_factor

    _if demand_kw _isnt _unset _andif pf _isnt _unset _andif pf > 0
    _then
        _self.apparent_power_kva << demand_kw / pf
        write("✓ Recalculated apparent power: ", _self.apparent_power_kva, " kVA")
    _endif

_endmethod
$
```

### Example 5: Status Propagation

```magik
_pragma(classify_level=basic, topic={business_rule})
_method eo_circuit.br_post_update_propagate_status()
    ## When circuit status changes, propagate to connected equipment

    _local old_rec << _self.old_version
    _if old_rec _is _unset _then _return _endif

    # Check if status changed
    _if old_rec.status = _self.status
    _then
        _return
    _endif

    write("Circuit status changed: ", old_rec.status, " → ", _self.status)

    # Get all equipment on this circuit
    _local equipment << _self.follows(:circuit_connectivity)

    _for equip _over equipment.fast_elements()
    _loop
        # Update equipment operational status based on circuit status
        _if _self.status = "de-energized"
        _then
            equip.operational_status << "out-of-service"
        _elif _self.status = "in-service"
        _then
            equip.operational_status << "in-service"
        _endif

        write("  ✓ Updated: ", equip.asset_id, " → ", equip.operational_status)
    _endloop

_endmethod
$
```

## Installation Script

```magik
_block
    ## Script to create POST UPDATE Business Rules

    _local post_update_rules << {
        {:br_xfrmr_update_bank, "eo_power_xfrmr", "br_post_update_recalculate_bank()"},
        {:br_pole_audit, "eo_pole", "br_post_update_audit_log()"},
        {:br_conductor_notify, "eo_conductor_inst", "br_post_update_notify_downstream()"},
        {:br_sp_calc_load, "eo_service_point", "br_post_update_calculate_load()"},
        {:br_circuit_propagate, "eo_circuit", "br_post_update_propagate_status()"}
    }

    _for rule_data _over post_update_rules.fast_elements()
    _loop
        _local (rule_name, table_name, method_name) << (_scatter rule_data)

        _local existing << sw_gis!business_rule.get_by_name(rule_name)
        _if existing _isnt _unset
        _then
            existing.delete()
        _endif

        sw_gis!business_rule.new_with(
            rule_name,
            table_name,
            "post",
            "update",
            method_name,
            _true
        )

        write("✓ POST UPDATE rule created: ", rule_name)
    _endloop

    gis_program_manager.databases[:gis].commit()
    write("Commit successful!")

_endblock
$
```

## Testing

### Test POST Update Rule

```magik
_block
    # Test audit log
    _local pole << eo_pole.select(predicate.eq(:asset_id, "TEST001")).an_element()

    _if pole _isnt _unset
    _then
        write("Original height: ", pole.pole_height)

        # Update
        pole.pole_height << 15.0

        # The POST UPDATE rule will execute automatically
        # Check the output for audit log

        write("New height: ", pole.pole_height)
    _endif

_endblock
```

## Best Practices

### ✅ Do

1. **Check if Field Actually Changed**
   ```magik
   _local old_rec << _self.old_version
   _if old_rec.field = _self.field _then _return _endif
   ```

2. **Use old_version for Comparisons**
   ```magik
   _local old_value << _self.old_version.status
   _local new_value << _self.status
   ```

3. **Protect Against Errors**
   ```magik
   _protect
       # your update logic
   _protection
       _handling error _with _proc(cond)
           write("Error in POST update: ", cond.report_contents_string)
       _endproc
   _endprotect
   ```

4. **Batch Related Updates**
   ```magik
   # Update all related objects in one go
   _for related_obj _over related_objects.fast_elements()
   _loop
       related_obj.some_field << new_value
   _endloop
   ```

### ❌ Avoid

1. ❌ **Don't Raise Conditions** (it's too late!)
   ```magik
   # Bad - too late to prevent update
   _method obj.br_post_update()
       condition.raise(:user_error, :string, "Invalid!")
   _endmethod
   ```

2. ❌ **Don't Create Infinite Loops**
   ```magik
   # Bad - modifying self triggers the same rule again!
   _method obj.br_post_update()
       _self.field << new_value  # INFINITE LOOP!
   _endmethod
   ```

3. ❌ **Don't Do Heavy Processing**
   ```magik
   # Bad - POST rules should be fast
   _method obj.br_post_update()
       _for obj _over huge_collection.all_elements()
       _loop
           # expensive operation
       _endloop
   _endmethod
   ```

## Common Use Cases

### 1. Audit Logging
Track changes to critical fields for compliance and troubleshooting.

### 2. Calculated Fields
Update dependent calculated fields when source fields change.

### 3. Status Propagation
Cascade status changes to related objects.

### 4. Cache Invalidation
Clear cached values when underlying data changes.

### 5. Notifications
Send notifications to external systems about changes.

### 6. Derived Relationships
Maintain relationships that are derived from other data.

## Performance Considerations

1. **Keep it Fast** - POST rules execute synchronously
2. **Limit Scope** - Only update what's necessary
3. **Batch Operations** - Group multiple updates together
4. **Avoid Recursion** - Don't modify the same object
5. **Consider Async** - Use background jobs for heavy work

## Debugging POST Rules

```magik
# Temporarily disable
sw_gis!business_rule.get_by_name(:br_your_rule).active? << _false

# Check old_version availability
_method obj.test_old_version()
    write("Old version: ", _self.old_version)
    write("Current value: ", _self.some_field)
    write("Old value: ", _self.old_version.some_field)
_endmethod

# Add debug logging
_method obj.br_post_update_with_logging()
    write("=== POST UPDATE TRIGGERED ===")
    write("Object: ", _self.asset_id)
    write("Old version exists? ", _self.old_version _isnt _unset)
    # ... rest of logic
_endmethod
```

## Next Steps

- [PRE Insert Validation](./PRE_Insert_Validation.md)
- [Duplicate Validation](./Duplicate_Validation.md)
- [Best Practices Guide](../Best_Practices_Guide.md)
