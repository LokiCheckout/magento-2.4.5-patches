# Patches for Magento 2.4.5-14

**PHP 8.1 supports various new features (like union types and `mixed`) and Magento 2.4.5 was meant to support them all properly. Unfortunately, Magento 2.4.5 contained serious bugs
that are fixed by upgrading to latest versions. This repository contains patches for Loki Checkout customers that are still kept on Magento 2.4.5.**

## Errors caused by this Magento 2.4.5 bug
The following errors are given when running `setup:di:compile`:

```bash
Type "mixed" cannot be nullable
```

And:

```bash
Call to undefined method ReflectionUnionType::getName()
```

## Possible solutions
1) Upgrade the core. The bug is in Magento 2.4.5 core. And not only the Loki extensions might suffer from this. Because of this, we strongly recommend you to upgrade to the latest Magento version.

2) To fix it permanently in the core, we recommend you to patch the core. The `mixed` error is fixed by an official quality patch ACSD-55031. The `ReflectionUnionType` error is fixed with the patch in this repository. For this specific error, a patch is provided in the `core/` folder of this repository.

3) Fix the issue per Loki extension with composer patches per extension. This sounds easier than patching the Magento core, but it is not. The composer patches are provided in the `module/` folder of this repository.

Do **NOT** use both patches from the `core/` folder and `module/` folder.

## Solution 1) Upgrade the core.
Drop it like it's hot.

## Solution 2) Patch the core.
This solution requires the following:

- Magento 2.4.5
- `cweagans/composer-patches` 1.0

### WARNING: Incompatible with Magento Quality Patch
The official Magento Quality Patch ACSD-55031 fixes the issue of the `mixed` error. Unfortunately, because Quality Patches are not delivered as composer patches and because both
fixes require modification of the same file, our patch approach is **not compatible** with the regular ACSD-55031 patch.

Instead, the ACSD-55031 patch is offered here as a composer patch instead. Other Quality Patches (that do not modify the files of these patches) should work fine.

### Usage
Copy the files `union-types-patch.diff` and `ACSD-55031_composer-patch.diff` contained in the `core/` folder of this repository into a folder `patches/` in your Magento project.

Run the following commands to install `cweagans/composer-patches`:
```bash
composer require cweagans/composer-patches:^1
```

Open up your Magento `composer.json` file and add the following:
```json
{
  "extra": {
    "patches": {
      "magento/framework": {
        "ACSD-55031 quality patch": "patches/ACSD-55031_composer-patch.diff",
        "Support union types": "patches/union-types-patch.diff"
      }
    }
  }
}
```

### Apply the patches
Re-run composer installation to apply these composer patches:

```bash
composer update
```

### End result
After applying both patches (using either scenario), the file `vendor/magento/framework/Interception/Code/Generator/Interceptor.php` should look like the following (with irrelevant parts left out):

```php
class Interceptor extends EntityAbstract
{
    /**
     * Get return type
     *
     * @param \ReflectionMethod $method
     * @return string|null
     */
    private function getReturnTypeValue(\ReflectionMethod $method): ?string
    {
        $returnTypeValue = null;
        $returnType = $method->getReturnType();
        if ($returnType) {
            if ($returnType instanceof \ReflectionUnionType || $returnType instanceof \ReflectionIntersectionType) {
                return $this->getReturnTypeValues($returnType, $method);
            }

            $className = $method->getDeclaringClass()->getName();
            $returnTypeValue = ($returnType->allowsNull() && $returnType->getName() !== 'mixed' ? '?' : '');
            $returnTypeValue .= ($returnType->getName() === 'self')
                ? $className ? '\\' . ltrim($className, '\\') : ''
                : $returnType->getName();
        }

        return $returnTypeValue;
    }

    /**
     * Get return type values for Intersection|Union types
     *
     * @param \ReflectionIntersectionType|\ReflectionUnionType $returnType
     * @param \ReflectionMethod $method
     * @return string|null
     */
    private function getReturnTypeValues(
        \ReflectionIntersectionType|\ReflectionUnionType $returnType,
        \ReflectionMethod $method
    ): ?string {
        $returnTypeValue = [];
        foreach ($method->getReturnType()->getTypes() as $type) {
            $returnTypeValue[] =  $type->getName();
        }

        return implode(
            $returnType instanceof \ReflectionUnionType ? '|' : '&',
            $returnTypeValue
        );
    }
}
```
## Solution 3) Patch the Loki Checkout extensions
This solution requires the following:

- Magento 2.4.5
- `cweagans/composer-patches` 1.0

### Usage
Copy all relevant `*.diff` files contained in the `modules/` folder of this repository into a folder `patches/` in your Magento project.

Run the following commands to install `cweagans/composer-patches`:
```bash
composer require cweagans/composer-patches:^1
```

Open up your Magento `composer.json` file and add something like the following:
```json
{
  "extra": {
    "patches": {
      "loki-checkout/magento2-core": {
        "Magento 2.4.5 patch": "patches/loki-checkout_magento2-core-2.4.5.diff"
      }
    }
  }
}
```

We have prepared a file `modules/patches.json` with this JSON for all available modules. If you want you can use this for the base of updating your own root `composer.json`.

```bash
cp composer.json composer-old.json
curl https://raw.githubusercontent.com/LokiCheckout/magento-2.4.5-patches/refs/heads/main/modules/patches.json -o loki-checkout-patches.json
jq -s add composer-old.json loki-checkout-patches.json > composer.json
```

### Apply the patches
Re-run composer installation to apply these composer patches.

```bash
composer update
```

### End result
After applying these patches, there should be no occurance of union types and the `mixed` keyword in the given sources.

