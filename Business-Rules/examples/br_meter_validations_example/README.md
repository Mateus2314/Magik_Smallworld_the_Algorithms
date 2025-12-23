# Business Rules Example - Pole Validations

Complete example module demonstrating **Business Rules implementation** for poles (`ed_pole`) in Smallworld GIS.

## Features

- 5 Business Rules (PRE/POST validations)
- Helper methods for data processing
- Installation/uninstallation scripts
- Automated tests

---

## Quick Start

### 1. Load Methods

```magik
load_file("path/to/source/business_rules/br_pole_validations.magik")
load_file("path/to/source/business_rules/br_pole_helpers.magik")
```

### 2. Install Business Rules

```magik
load_file("path/to/scripts/install/insert_br_pole.magik")
```

### 3. Run Tests

```magik
load_file("path/to/scripts/tests/test_pole_rules.magik")
```

---

## Business Rules Overview

| Rule | Type | Description |
|------|------|-------------|
| PRE INSERT | Validation | Validates geometry, height (6-20m), and distance (min 5m) |
| PRE UPDATE | Validation | Validates height and structural capacity |
| PRE DELETE | Validation | Prevents deletion if equipment is connected |
| POST INSERT | Log | Records pole creation details |
| POST UPDATE | Log | Records pole modification details |

---

## Helper Methods

| Method | Description |
|--------|-------------|
| `buscar_equipamentos_conectados()` | Search for connected equipment |
| `calcular_carga_total_poste()` | Calculate total installed load |
| `gerar_relatorio_validacao()` | Generate validation report |
| `buscar_postes_proximos(distance)` | Search nearby poles within distance |
| `validar_capacidade_estrutural_simplificada()` | Validate structural capacity |

---

## Manual Testing

### PRE INSERT
```magik
# Test 1: Missing geometry
insert_pole_without_geometry()  # Expected: ERROR

# Test 2: Invalid height
insert_pole_with_height(3)  # Expected: ERROR (< 6m)

# Test 3: Valid pole
insert_pole_with_height(9)  # Expected: SUCCESS + log
```

### PRE UPDATE
```magik
# Test 1: Height too high
update_pole_height(25)  # Expected: ERROR (> 20m)

# Test 2: Valid update
update_pole_height(12)  # Expected: SUCCESS + log
```

### PRE DELETE
```magik
# Test 1: With equipment
delete_pole_with_equipment()  # Expected: ERROR

# Test 2: Empty pole
delete_empty_pole()  # Expected: Confirmation dialog
```

---

## Configuration

Business Rules are stored in `cs_config` database:

```magik
dataset_name: "electric"
collection_name: "ed_pole"
rule_level: "Object"
mandatory_ind: _true
enabled_ind: _true
```

---

## Troubleshooting

**Methods not found:**
```magik
# Ensure methods are loaded before installing BRs
load_file("...br_pole_validations.magik")
load_file("...br_pole_helpers.magik")
```

**BRs not executing:**
- Verify methods are loaded
- Check BRs are enabled in `cs_config`
- Confirm dataset/collection names match

**Stale record handle error:**
```magik
# Run uninstall twice
load_file("...remove_br_pole.magik")
load_file("...remove_br_pole.magik")
```

---

## Project Structure

```
br_meter_validations_example/
├── README.md
├── module.def
├── load_list.txt
├── source/business_rules/
│   ├── br_pole_validations.magik
│   ├── br_pole_helpers.magik
│   └── load_list.txt
└── scripts/
    ├── install/insert_br_pole.magik
    ├── tests/test_pole_rules.magik
    └── uninstall/remove_br_pole.magik
```

---

## References

- [Best Practices Guide](../../Best_Practices_Guide.md)
- [PRE Insert Validation Example](../PRE_Insert_Validation.md)
- [POST Update Action Example](../POST_Update_Action.md)

---

## License

Educational purposes only. Use at your own risk in production environments.

---

**Author:** Mateus De La Fuente Cezar | **Year:** 2025
