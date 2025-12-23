# PRE Insert Validation

## Objective

Validate data **before** inserting a new record, preventing invalid data from entering the database.

---

## When to Use

Use PRE Insert validations when you need to:

- ✅ Enforce **mandatory fields**
- ✅ Validate **data ranges** (min/max values)
- ✅ Check **spatial constraints** (distance, overlap)
- ✅ Verify **business logic** (complex rules)
- ✅ **Prevent duplicates**

---

## How It Works

```
User Action          Business Rule          Database
    |                      |                    |
    | Insert record        |                    |
    |--------------------->|                    |
    |                      |                    |
    |                      | Validate data      |
    |                      |                    |
    |                      | Valid?             |
    |                      |  ├─ YES: Continue  |
    |                      |  |    |            |
    |                      |  |    |----------->| INSERT
    |                      |  |                 |
    |                      |  └─ NO: BLOCK      |
    |<---------------------|    Show error      |
    | ERROR message        |                    |
```

---

## Configuration in cs_config

```magik
Business Rule:
  name: "PRE INSERT Pole Validation"
  run_method: "br_pre_insert_validacao_basica()"

Business Rule Usage:
  dataset_name: "electric"
  collection_name: "ed_pole"
  rule_type: "Pre-Insert Action"
  rule_level: "Object"
  sequence_number: 100
  mandatory_ind: _true
  enabled_ind: _true
```

---

## Complete Example - Pole Validation

### Implementation

```magik
_method ed_pole.br_pre_insert_validacao_basica()
    ## PRE INSERT - Validates geometry, height, and distance

    # ========================================
    # VALIDATION 1: Geometry is mandatory
    # ========================================
    _if _self.location _is _unset
    _then
        condition.raise(:user_error,
                :string, "ERROR: Pole geometry must be defined!",
                :pole, _self)
    _endif

    # ========================================
    # VALIDATION 2: Height range (6-20m)
    # ========================================
    _local altura << _self.altura_util_m

    _if altura _is _unset
    _then
        condition.raise(:user_error,
                :string, "ERROR: Pole height must be filled!",
                :pole, _self)
    _endif

    _if altura < 6 _orif altura > 20
    _then
        condition.raise(:user_error,
                :string, write_string("ERROR: Height must be between 6-20m. Given: ",
                                     altura, "m"),
                :pole, _self)
    _endif

    # ========================================
    # VALIDATION 3: Minimum distance (5m)
    # ========================================
    _local nearby_poles << _self.buscar_postes_proximos(5)

    _if _not nearby_poles.empty?
    _then
        _local dist << nearby_poles.an_element()[:distance]
        condition.raise(:user_error,
                :string, write_string("ERROR: Another pole at ",
                                     dist.rounded, "m. Minimum: 5m"),
                :pole, _self,
                :nearby_poles, nearby_poles)
    _endif

    # All validations passed
    write("OK: Pole passed PRE INSERT validations")

_endmethod
```

### Helper Method - Search Nearby Poles

```magik
_method ed_pole.buscar_postes_proximos(max_distance)
    ## Search for nearby poles within specified distance
    ##
    ## Parameters:
    ##   max_distance: float - distance in meters
    ##
    ## Returns:
    ##   rope of property_list with :pole and :distance

    _local results << rope.new()

    _if _self.location _is _unset
    _then
        _return results
    _endif

    # Create search area
    _local search_area << _self.location.bounds.buffered_by(max_distance)

    # Query poles in area
    _local poles << _self.source_view.world.geometry_at_bounds(
        search_area,
        ed_pole.collection_name)

    # Filter by distance
    _for other_pole _over poles.fast_elements()
    _loop
        _if other_pole = _self _then _continue _endif

        _local dist << _self.location.distance_to(other_pole.location)

        _if dist <= max_distance
        _then
            results.add_last(property_list.new_with(
                :pole, other_pole,
                :distance, dist))
        _endif
    _endloop

    _return results

_endmethod
```

---

## Testing

### Test 1: Missing Geometry

```magik
# Create pole without geometry
_local pole << ed_pole.new_detached()
pole.altura_util_m << 10
pole.insert()

# Expected Result:
# ERROR: Pole geometry must be defined!
```

### Test 2: Invalid Height

```magik
# Create pole with height < 6m
_local pole << ed_pole.new_detached()
pole.location << coordinate(100, 100)
pole.altura_util_m << 3
pole.insert()

# Expected Result:
# ERROR: Height must be between 6-20m. Given: 3m
```

### Test 3: Too Close to Another Pole

```magik
# Insert first pole
_local pole1 << ed_pole.new_detached()
pole1.location << coordinate(100, 100)
pole1.altura_util_m << 10
pole1.insert()

# Try to insert second pole at 3m distance
_local pole2 << ed_pole.new_detached()
pole2.location << coordinate(103, 100)
pole2.altura_util_m << 10
pole2.insert()

# Expected Result:
# ERROR: Another pole at 3m. Minimum: 5m
```

### Test 4: Valid Insertion

```magik
# Insert pole with valid data
_local pole << ed_pole.new_detached()
pole.location << coordinate(200, 200)
pole.altura_util_m << 12
pole.insert()

# Expected Result:
# OK: Pole passed PRE INSERT validations
# Record inserted successfully
```

---

## Installation

### 1. Load Validation Method

```magik
load_file("path/to/br_pole_validations.magik")
load_file("path/to/br_pole_helpers.magik")
```

### 2. Create Business Rule

```magik
_local csv << gis_program_manager.cached_dataset(:cs_config)
_local brt << csv.collection(:swo_business_rule)
_local brut << csv.collection(:swo_business_rule_usage)

csv.switch(:write)

# Create Business Rule
_local br << brt.new_detached_record().prepare_fields_and_values()
br[:run_method] << "br_pre_insert_validacao_basica()"
br[:name] << "PRE INSERT Pole Validation"
br[:description] << "Validates geometry, height and distance"
_local business_rule << record_transaction.new_insert(brt, br).run()

# Create Usage
_local usage << brut.new_detached_record().prepare_fields_and_values()
usage[:dataset_name] << "electric"
usage[:collection_name] << "ed_pole"
usage[:rule_level] << "Object"
usage[:sequence_number] << 100
usage[:mandatory_ind] << _true
usage[:enabled_ind] << _true
usage[:business_rule] << business_rule
usage[:rule_type] << "Pre-Insert Action"
record_transaction.new_insert(brut, usage).run()

csv.commit()
csv.switch(:readonly)

write("PRE INSERT Business Rule installed successfully!")
```

---

## Best Practices

### ✅ DO

- **Validate early**: Check critical fields first
- **Clear messages**: Explain what's wrong and how to fix it
- **Use helpers**: Encapsulate complex logic in separate methods
- **Test thoroughly**: Cover all validation paths
- **Document**: Explain business rules and constraints

### ❌ DON'T

- **Don't modify data**: PRE validations should only check, not change
- **Don't query excessively**: Cache repeated queries
- **Don't catch all errors**: Let validation errors propagate
- **Don't hardcode values**: Use configuration or parameters
- **Don't skip edge cases**: Test boundary conditions

---

## Common Patterns

### Pattern 1: Mandatory Field

```magik
_if _self.required_field _is _unset
_then
    condition.raise(:user_error,
            :string, "ERROR: Field 'required_field' is mandatory!")
_endif
```

### Pattern 2: Range Validation

```magik
_if value < min _orif value > max
_then
    condition.raise(:user_error,
            :string, write_string("ERROR: Value must be between ",
                                 min, " and ", max))
_endif
```

### Pattern 3: Spatial Constraint

```magik
_local nearby << _self.find_nearby_objects(min_distance)
_if _not nearby.empty?
_then
    condition.raise(:user_error,
            :string, "ERROR: Too close to existing objects!")
_endif
```

### Pattern 4: Business Logic

```magik
_if _self.status = "active" _andif _self.end_date _isnt _unset
_then
    condition.raise(:user_error,
            :string, "ERROR: Active records cannot have end date!")
_endif
```

---

## Complete Working Example

For a complete, tested implementation, see:
[br_meter_validations_example](br_meter_validations_example/)

This example includes:
- ✅ Full source code with validation methods
- ✅ Helper methods for complex logic
- ✅ Installation scripts
- ✅ Automated test suite
- ✅ Uninstallation scripts
- ✅ Comprehensive documentation

---

## Related Examples

- [PRE Update Validation](PRE_Update_Validation.md) - Validate before updates
- [PRE Delete Validation](PRE_Delete_Validation.md) - Validate before deletion
- [POST Insert Action](POST_Insert_Action.md) - Actions after insertion
- [Duplicate Validation](Duplicate_Validation.md) - Prevent duplicates

---

## References

- Smallworld Business Rules API Documentation
- [Best Practices Guide](../Best_Practices_Guide.md)
- [Architecture Overview](../../Architecture/product-structure.md)
