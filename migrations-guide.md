# Migrations Guide

[Back to README](./README.md)

## Quick Start

[`coding-style.md`](./coding-style.md) | [`architecture-style.md`](./architecture-style.md) | [`logging-guide.md`](./logging-guide.md) | [`unit-testing-guide.md`](./unit-testing-guide.md) | [`migrations-guide.md`](./migrations-guide.md) | [`rbac-standards-guide.md`](./rbac-standards-guide.md)

This guide explains how to write and run Directus schema migrations.

It is based on the existing `.migrations` workspace and scripts already used in NewMoon repository.

## 1) What migrations are for

Use migrations for **schema changes** in Directus:

- creating collections/fields/relations
- changing field meta/options
- updating permissions or schema-level structures
- adding seed data like defaults, labels, etc...

Do not use migrations for bulk content setup. Use seeds for that.

## 2) Where migrations live

- Workspace: `.migrations`
- Migration files: `.migrations/migrations/*.js` 
- Tracking collection in Directus: `schema_migrations`

Applied migration filenames are stored in `schema_migrations`, so each migration runs once per environment.

## 3) Setup

From `.migrations`, create `.env` from example and configure:

```env
DIRECTUS_URL=http://localhost:8055
DIRECTUS_TOKEN=your_admin_token
```

Then initialize migration tracking collection (once per environment):

```bash
pnpm setup:schema
```

Full setup script listing (`.migrations/setup-migration-schema.js`):

```js
import { createDirectus, rest, staticToken, createCollection, createField } from '@directus/sdk';
import 'dotenv/config';

const client = createDirectus(process.env.DIRECTUS_URL)
  .with(staticToken(process.env.DIRECTUS_TOKEN))
  .with(rest());

async function ensureMigrationCollection() {
  try {
    // Check if collection exists
    let collectionExists = false;
    try {
      const collections = await client.request(({ collections }) =>
        collections.readByQuery({ filter: { collection: { _eq: 'schema_migrations' } } })
      );
      if (collections?.data?.length > 0) collectionExists = true;
    } catch (err) {
      // Fallback for older SDK versions, do nothing, will try to create if not found
      collectionExists = false;
    }

    if (!collectionExists) {
      try {
        await client.request(
          createCollection({
            collection: 'schema_migrations',
            schema: {},
            meta: {
              icon: 'schema',
              note: 'Tracks applied migrations',
            },
            fields: [
              {
                field: 'id',
                type: 'uuid',
                schema: {
                  is_primary_key: true,
                  has_auto_increment: false,
                },
                meta: {
                  interface: 'input',
                  special: ['uuid'],
                  hidden: true,
                },
              },
            ],
          })
        );
        console.log('Collection "schema_migrations" created with UUID primary key.');
      } catch (err) {
        // If collection was created in the meantime, ignore error
        if (
          err?.errors?.some(
            (e) =>
              e?.message?.toLowerCase().includes('already exists') ||
              e?.extensions?.code === 'COLLECTION_ALREADY_EXISTS'
          )
        ) {
          console.log('Collection "schema_migrations" already exists.');
        } else {
          throw err;
        }
      }
    } else {
      console.log('Collection "schema_migrations" already exists.');
    }

    const extraFields = [
      {
        field: 'filename',
        type: 'string',
        schema: { is_nullable: false, is_unique: true },
        meta: { required: true, interface: 'input', display: 'Migration Filename' },
      },
      {
        field: 'applied_at',
        type: 'timestamp',
        schema: { is_nullable: false },
        meta: { required: true, interface: 'datetime', display: 'Applied At' },
      },
    ];

    for (const field of extraFields) {
      try {
        await client.request(createField('schema_migrations', field));
        console.log(`Field "${field.field}" created`);
      } catch (err) {
        if (err?.errors?.[0]?.extensions?.code === 'FIELD_ALREADY_EXISTS' ||
            err?.errors?.some(e =>
              e?.message?.toLowerCase().includes('already exists') ||
              e?.extensions?.code === 'FIELD_ALREADY_EXISTS'
            )) {
          console.log(`Field "${field.field}" already exists.`);
        } else {
          throw err;
        }
      }
    }

    console.log('"schema_migrations" collection is ready!');
  } catch (error) {
    console.error('Failed to create collection or fields:', error.message);
    process.exit(1);
  }
}

ensureMigrationCollection();
```

## 4) Create a migration file

Naming format:

`NNN_TICKET_description.js`

Example:

`042_UXUI-7000_add_service_priority_field.js`

Rules:

- Export `up(client)` function.
- Keep one logical change per file.
- Make it idempotent.
- Keep logs clear and simple.
- Never rewrite old migrations.

## 5) Example migration

```js
import { createField, readField } from '@directus/sdk';

export async function up(client) {
  try {
    await client.request(readField('services', 'priority'));
    console.log('services.priority exists, skipping');
    return;
  }
  catch (error) {
    const code = error?.errors?.[0]?.extensions?.code;
    const notFound = code === 'FIELD_NOT_FOUND' || error?.response?.status === 404;
    if (!notFound) throw error;
  }

  await client.request(createField('services', {
    field: 'priority',
    type: 'integer',
    meta: {
      interface: 'input',
      required: false,
      note: 'Sorting priority for services',
    },
    schema: {
      is_nullable: true,
    },
  }));

  console.log('Created services.priority');
}
```

## 6) Run migrations

From `.migrations`:

```bash
pnpm migrate
```

Full migrate script listing (`.migrations/migrate.js`):

```js
import { createDirectus, rest, staticToken, readCollection, readItems, createItem } from '@directus/sdk';
import { readdir } from 'fs/promises';
import { join, dirname } from 'path';
import { fileURLToPath } from 'url';
import 'dotenv/config';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const DIRECTUS_URL = process.env.DIRECTUS_URL;
const DIRECTUS_TOKEN = process.env.DIRECTUS_TOKEN;
const MIGRATIONS_COLLECTION = 'schema_migrations';

if (!DIRECTUS_URL || !DIRECTUS_TOKEN) {
  console.error('Error: DIRECTUS_URL and DIRECTUS_TOKEN must be set in environment variables');
  process.exit(1);
}

const client = createDirectus(DIRECTUS_URL)
  .with(staticToken(DIRECTUS_TOKEN))
  .with(rest());

/**
 * Check if a collection exists
 */
async function collectionExists(collectionName) {
  try {
    await client.request(readCollection(collectionName));
    return true;
  } catch (error) {
    const code = error?.errors?.[0]?.extensions?.code;
    const status = error?.response?.status;
    const isNotFound = code === 'COLLECTION_NOT_FOUND' || status === 404;
    if (isNotFound) return false;
    const message = `Unexpected error while checking collection "${collectionName}": ${error.message}`;
    console.error(message);
    throw new Error(message);
  }
}

/**
 * Get applied migration filenames from the migrations collection
 */
async function getAppliedMigrations() {
  try {
    if (!(await collectionExists(MIGRATIONS_COLLECTION))) {
      throw new Error(`Collection "${MIGRATIONS_COLLECTION}" does not exist. Run "pnpm setup:schema" first.`);
    }

    const items = await client.request(
      readItems(MIGRATIONS_COLLECTION, {
        limit: -1,
        fields: ['filename']
      })
    );

    return items.map(item => item.filename);
  } catch (error) {
    console.error(`Error reading applied migrations: ${error.message}`);
    throw error;
  }
}

/**
 * Record a migration as applied
 */
async function recordMigration(migration) {
  try {
    await client.request(
      createItem(MIGRATIONS_COLLECTION, {
        filename: migration.filename,
        applied_at: new Date().toISOString(),
      })
    );
  } catch (error) {
    console.error(`Error recording migration ${migration.filename}: ${error.message}`);
    throw error;
  }
}

async function loadMigrations() {
  const migrationsDir = join(__dirname, 'migrations');

  try {
    const files = await readdir(migrationsDir);

    const migrationFiles = files
      .filter(file => /^\d+_.+\.js$/.test(file))
      .sort();

    const migrations = [];

    for (const file of migrationFiles) {
      const filePath = join(migrationsDir, file);
      const module = await import(filePath);

      const migrationId = file;
      const upFunction = module.up || module.default?.up;

      if (!upFunction) {
        console.warn(`Warning: ${file} is missing 'up' export, skipping`);
        continue;
      }

      migrations.push({
        file,
        id: migrationId,
        filename: file,
        up: upFunction
      });
    }

    return migrations;
  } catch (error) {
    if (error.code === 'ENOENT') {
      console.error('Error: migrations directory not found');
      return [];
    }
    throw error;
  }
}

async function migrate() {
  console.log('Starting migration process...\n');

  try {
    console.log('Checking applied migrations...');
    const appliedMigrations = await getAppliedMigrations();
    console.log(`   Found ${appliedMigrations.length} previously applied migration(s)\n`);

    console.log('Loading migration files...');
    const migrations = await loadMigrations();
    console.log(`   Found ${migrations.length} migration file(s)\n`);

    if (migrations.length === 0) {
      console.log('No migrations to run');
      return;
    }

    let appliedCount = 0;
    let skippedCount = 0;
    let errorCount = 0;

    for (const migration of migrations) {
      if (appliedMigrations.includes(migration.filename)) {
        console.log(`Skipping ${migration.filename} (${migration.id}) - already applied`);
        skippedCount++;
        continue;
      }

      try {
        console.log(`Applying ${migration.filename} (${migration.id})...`);

        await migration.up(client);
        await recordMigration(migration);

        console.log(`Applied ${migration.filename} (${migration.id})`);
        appliedCount++;
      } catch (error) {
        console.error(`Error applying ${migration.filename} (${migration.id}):`);
        console.error(`   ${error.message}`);
        if (error.stack) {
          console.error(`   Stack: ${error.stack}`);
        }
        errorCount++;
        break;
      }
    }

    console.log('\n' + '='.repeat(50));
    console.log('Migration Summary:');
    console.log(`   Applied: ${appliedCount}`);
    console.log(`   Skipped: ${skippedCount}`);
    console.log(`   Errors: ${errorCount}`);
    console.log('='.repeat(50));

    if (errorCount > 0) {
      process.exit(1);
    } else {
      console.log('\nMigration process completed successfully!');
    }
  } catch (error) {
    console.error('\nFatal error during migration:');
    console.error(error);
    process.exit(1);
  }
}

migrate();
```

Behavior:

- validates required env vars
- checks that `schema_migrations` exists
- runs pending files in lexicographic order
- records filename + applied time on success
- stops on first error

## 7) Seeds vs migrations

Use migrations for schema, seeds for demo or initial content.

Available scripts in `.migrations`:

```bash
pnpm seed
pnpm cleanup:seed
```

## 8) Safe workflow for teams

1. Pull latest main branch.
2. Create one new migration file per ticket/change.
3. Run migration locally.
4. Verify in Directus UI/API that schema is correct
5. Commit migration with clear message.
6. In PR description, include:
  - migration filename
  - expected schema result
  - rollback strategy (if needed)

## 9) Rollback strategy

Current runner is `up`-only. For rollback:

- create a new migration that reverts the previous schema change, or
- manually revert only in emergency and immediately codify revert migration

Never delete migration history rows from `schema_migrations` in shared environments.