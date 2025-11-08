<p align="center"><a href="https://fourdotsix.com" target="_blank"><img src="https://cdn.fourdotsix.com/images/4.6/logo/4.6-bimi.svg" width="200" alt="fourdotsix.com Logo"></a><br><strong>four●six</strong> // tenant-buckets</p>

# Provision S3 Buckets for each tenant

[![Latest Version on Packagist](https://img.shields.io/packagist/v/vidwan/tenant-buckets.svg?style=flat-square)](https://packagist.org/packages/vidwan/tenant-buckets)
[![Tests](https://github.com/vidwanco/tenant-buckets/actions/workflows/run-tests.yml/badge.svg?branch=main)](https://github.com/vidwanco/tenant-buckets/actions/workflows/run-tests.yml)
[![Total Downloads](https://img.shields.io/packagist/dt/vidwan/tenant-buckets.svg?style=flat-square)](https://packagist.org/packages/vidwan/tenant-buckets)

Automatically Provision AWS S3 Buckets for each tenant. It's an Extension for [stancl/tenancy](https://github.com/stancl/tenancy). For more details refer to [TenancyForLaravel](https://tenancyforlaravel.com/).

## Tenancy V4.x

Use branch [tenancy-v4](https://github.com/vidwanco/tenant-buckets/tree/tenancy-v4) for Tenancy v4.x versions.

Use `dev-tenancy-v4` for pulling latest changes from branch.

```bash
composer require vidwan/tenant-buckets:dev-tenancy-v4
```

Or just use a Release Candidate:

```bash
composer require vidwan/tenant-buckets:^4.0.0-rc
```

## Concept

The concept is simple, to automatically provision a new AWS S3 bucket for tenant on registration and update the same on the central database's tenant table & data column under `tenant_bucket`.
Then using a bootstrapper updating the bucket in config `filesystems.disks.s3.bucket` during runtime when in Tenant's context and then reverting it back on central context.

## Installation

You can install the package via composer:

```bash
composer require vidwan/tenant-buckets
```

## Usage

### 1. Filesystem Config Setup

Ensure your S3 configuration has all the Key/Value pairs, as below:

**File:** `config/filesystems.php` 
```php
        's3' => [
            'driver' => 's3',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'bucket' => env('AWS_BUCKET'),
            'url' => env('AWS_URL'),
            'endpoint' => env('AWS_ENDPOINT'),
            'use_path_style_endpoint' => env('AWS_USE_PATH_STYLE_ENDPOINT', false),
        ],

```
> Using Minio for development? Make sure to update your `.env` with `AWS_USE_PATH_STYLE_ENDPOINT=true`

### 2. Tenancy Config

There are two parts in tenancy config to take care of.
#### Part **a**.

Add the `TenantBucketBootstrapper::class` to the tenancy config file under `bootstrappers`.

```php
Vidwan\TenantBuckets\Bootstrappers\TenantBucketBootstrapper::class
```

**File:** `config/tenancy.php`
```php
    'bootstrappers' => [
        // Tenancy Bootstrappers
        Vidwan\TenantBuckets\Bootstrappers\TenantBucketBootstrapper::class,
    ],
```

#### Part **b**.

Make sure the `s3` is commented in `tenancy.filesystem.disks` config or else it conflicts with the tenancy itself.

**File:** `config/tenancy.php`
```php
    'filesystem' => [
        'suffix_base' => 'tenant',
        'disks' => [
            'local',
            'public',
            // 's3', // Make sure this stays commented
        ],
    ],
```

### 3. Job Pipeline

Add `Vidwan\TenantBuckets\Jobs\CreateTenantBucket` & `Vidwan\TenantBuckets\Jobs\DeleteTenantBucket` in `JobPipeline::make()`. As the name suggests, the former Creates a New Bucket on Tenant Creation and the later Deletes it when a Tenant is being Deleted.

**File:** `app/Providers/TenancyServiceProviders.php`
```php

use Vidwan\TenantBuckets\Jobs\CreateTenantBucket;
use Vidwan\TenantBuckets\Jobs\DeleteTenantBucket;

...

    public function events()
    {
        return [
            // Tenant events
            ...
            Events\TenantCreated::class => [
                JobPipeline::make([
                    Jobs\CreateDatabase::class,
                    Jobs\MigrateDatabase::class,
                    Jobs\SeedDatabase::class,
                    // Your own jobs to prepare the tenant.
                    // Provision API keys, create S3 buckets, anything you want!
                    CreateTenantBucket::class, // <-- Place it Here
					
                ])->send(function (Events\TenantCreated $event) {
                    return $event->tenant;
                })->shouldBeQueued(false),
            ],
            ...
            Events\DeletingTenant::class => [
                JobPipeline::make([
                    DeleteTenantBucket::class, // <-- Place it Here
                ])->send(function (Events\DeletingTenant $event) {
                    return $event->tenant;
                })->shouldBeQueued(false),
            ],
            ...
        ];
    }
```

Cheers! 🥳
## Testing

```bash
composer test
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Contributing

Please see [CONTRIBUTING](.github/CONTRIBUTING.md) for details.

## Security Vulnerabilities

Please review [our security policy](../../security/policy) on how to report security vulnerabilities.

## Credits

- [Shashwat Mishra](https://github.com/secrethash)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.

## <br><br>

<p align="center">From the folks at <a href="https://fourdotsix.com" target="_blank">Four Dot Six (4.6)</a></p>
