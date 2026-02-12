[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Introduction

This is a plugin for [InvenTree](https://github.com/inventree/InvenTree/) that automatically generates **Still Asking Part Numbers (SAPN)** when parts are created.

The SAPN format is: `SAPN-{CCC}-{SS}-{NNNNN}`

- **CCC**: 3-letter category code (from part parameter `SA_CCC`)
- **SS**: 2-digit subcategory code (from part parameter `SA_SS`)
- **NNNNN**: 5-digit zero-padded sequence number, unique per (CCC, SS) bucket

**Example**: `SAPN-ELC-11-00042`

## How It Works

1. When a part is created (or changed, if enabled), the plugin checks for the `SA_CCC` and `SA_SS` part parameters
2. These parameters should be inherited from the part's category (typically set up using InvenTree's parameter templates)
3. The plugin finds the highest existing IPN with the matching prefix and increments the sequence
4. The new IPN is assigned to the part automatically

### Important Notes

- **CCC and SS always come from part parameters**, not from category names or paths
- Parts in deeply nested categories work correctly as long as the parameters are inherited
- Each (CCC, SS) combination has its own independent sequence starting at 00001
- The sequence maximum is 99999; the plugin will **error out** (not wrap around) if this limit is reached

## Installation

### Method 1: Using plugins.txt (Recommended for Docker)

Add the following line to your `plugins.txt` file:

```
git+https://github.com/still-asking/inventree-sapn-generator@main
```

Then run the InvenTree plugin install routine:

```bash
invoke install
```

### Method 2: Direct pip install

```bash
pip install git+https://github.com/still-asking/inventree-sapn-generator@main
```

### Enabling the Plugin

1. In InvenTree, go to **Settings** → **Plugin Settings**
2. Enable **Event Integration** (required for the plugin to receive events)
3. Find "SAPN Generator" in the plugin list and enable it
4. Configure the plugin settings as needed

## Settings

| Setting       | Description                                             | Default |
| ------------- | ------------------------------------------------------- | ------- |
| **Active**    | Master toggle for the plugin                            | `True`  |
| **On Create** | Generate SAPN when creating new parts                   | `True`  |
| **On Change** | Generate SAPN when editing parts (only if IPN is empty) | `False` |

## Setup Requirements

### 1. Create Parameter Templates

Create two parameter templates in InvenTree:

- **SA_CCC**: For the 3-letter category code (e.g., `ELC`, `MEC`, `PCB`)
- **SA_SS**: For the 2-digit subcategory code (e.g., `11`, `22`, `01`)

### 2. Assign Parameters to Categories

For each part category that should use auto-generated IPNs:

1. Go to the category
2. Add the `SA_CCC` parameter with the appropriate 3-letter code
3. Add the `SA_SS` parameter with the appropriate 2-digit code

### 3. Enable Parameter Inheritance

Ensure that parts in these categories inherit the parameters from their category. InvenTree can be configured to copy category parameters to new parts automatically.

## Validation Rules

The plugin validates the parameters before generating an IPN:

| Parameter | Pattern      | Valid Examples      | Invalid Examples           |
| --------- | ------------ | ------------------- | -------------------------- |
| SA_CCC    | `^[A-Z]{3}$` | `ELC`, `MEC`, `ABC` | `elc`, `AB`, `ABCD`, `A1C` |
| SA_SS     | `^\d{2}$`    | `00`, `11`, `99`    | `1`, `123`, `AB`           |

If either parameter is missing or invalid, the plugin will log a warning and skip IPN generation for that part.

## Concurrency Handling

The plugin includes a retry mechanism to handle race conditions when multiple parts are created simultaneously in the same (CCC, SS) bucket:

- Up to 10 retry attempts on integrity errors
- Each retry recomputes the next sequence number from the current database state
- Uses atomic transactions for safe updates

## Overflow Protection

The sequence number has a maximum of 99999. If a (CCC, SS) bucket reaches this limit:

- **The plugin will NOT wrap around** to avoid duplicate IPNs
- An error will be logged
- The IPN will not be assigned

This is intentional behavior to prevent duplicate IPN issues. If you reach this limit, consider:

- Creating a new subcategory (different SS code)
- Archiving old parts that are no longer needed

## Examples

### Category Structure Example

```
Electronics (SA_CCC=ELC)
├── Resistors (SA_SS=11)
│   └── SMD Resistors
│       └── 0805 Package
├── Capacitors (SA_SS=12)
└── ICs (SA_SS=21)

Mechanical (SA_CCC=MEC)
├── Fasteners (SA_SS=11)
└── Enclosures (SA_SS=21)
```

### Generated IPNs

| Part Category   | SA_CCC | SA_SS | Generated IPN     |
| --------------- | ------ | ----- | ----------------- |
| First resistor  | ELC    | 11    | SAPN-ELC-11-00001 |
| Second resistor | ELC    | 11    | SAPN-ELC-11-00002 |
| First capacitor | ELC    | 12    | SAPN-ELC-12-00001 |
| First fastener  | MEC    | 11    | SAPN-MEC-11-00001 |
| Third resistor  | ELC    | 11    | SAPN-ELC-11-00003 |

## Troubleshooting

### IPN not generated

1. Check that the plugin is enabled and "Active" is on
2. Verify "On Create" is enabled for new parts
3. Check that the part has both `SA_CCC` and `SA_SS` parameters with valid values
4. Review InvenTree logs for warnings from "SAPN Generator"

### Parameters not appearing on parts

1. Ensure parameter templates exist for `SA_CCC` and `SA_SS`
2. Check that category parameters are set correctly
3. Verify InvenTree is configured to inherit parameters from categories

### Duplicate IPNs

This should not happen if InvenTree has unique IPN enforcement enabled. If you see duplicates:

1. Check if InvenTree's "Enforce unique IPN" setting is enabled
2. The plugin includes retry logic to prevent race conditions
3. Review logs for any error messages

## Development

### Running Tests

```bash
cd ipn_generator/tests
python -m pytest test_sapn_generator.py -v
```

### Project Structure

```
inventree-sapn-generator/
├── ipn_generator/
│   ├── __init__.py
│   ├── generator.py          # Main plugin code
│   └── tests/
│       ├── __init__.py
│       ├── test_IPNGenerator.py    # Original tests (legacy)
│       └── test_sapn_generator.py  # SAPN-specific tests
├── pyproject.toml
├── README.md
└── LICENSE
```

## License

MIT License - see [LICENSE](LICENSE) for details.

## Credits

Based on [inventree-ipn-generator](https://github.com/LavissaWoW/inventree-ipn-generator)
