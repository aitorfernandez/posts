# "Bad" Habits

Recently, I've been thinking on the habits I consistently repeat when I'm coding. While they may not necessarily be inherently bad, they are likely to generate varying opinions. Let's dive into some of them:

## Migrations

All migrations are in lowercase. Lowercase is more natural and makes them easier to read, as they tend to blend with other parts of the code. Additionally, all IDEs highlight the keywords, and, come on, we're not in the eighties to require uppercase.

```sql
create table if not exists profile (
    id varchar primary key,
    full_name varchar not null
);
```

## Singular

As seen in the previous query, I prefer to use singular table names. It feels more natural when writing queries.

```sql
select full_name from profile where ...;
```

## Object column

All my tables include the column 'object'. This habit may seem somewhat redundant and I can agree with that sentiment. However, it has proven to be immensely helpful during debugging and handling multiple payloads in responses, while also making public interfaces more human-readable.

```sql
create type profile_object as enum ('profile');

create table if not exists profile (
  id varchar primary key,
  full_name varchar not null,
  object profile_object not null default 'profile'::profile_object,
);
```

## Default values

Prepare the database to handle additional tasks beyond simple data storage. This allows me to focus more on application logic and worry less about certain data.

```sql
create or replace function trigger_set_updated_at() returns trigger as $$
begin
  new.updated_at := current_timestamp;
  return new;
end;
$$ language plpgsql;

create table if not exists profile (
  id varchar primary key,
  full_name varchar not null,
  object profile_object not null default 'profile'::profile_object,
  updated_at timestamptz not null default current_timestamp
);

create trigger set_updated_at before update on profile_log
  for each row execute procedure trigger_set_updated_at();

-- Or if you are a fan of uuid
create table if not exists profile (
    id uuid primary key default gen_random_uuid(),
    -- [...]
);
```

## Models

Each layer in the application has its own models, working with data in the way that each layer requires. This approach ensures that each layer performs the specific tasks needed for its functionality, without affecting other layers.

```rust
// app layer
use async_graphql::{InputObject, ID};
#[derive(InputObject)]
struct ProfileInput {
    id: ID,
    full_name: String,
}

// domain layer
use chrono::{DateTime, Utc};

#[derive(Debug, Eq, PartialEq)]
pub enum ProfileObject {
    Profile,
}

pub struct Profile {
    pub id: String,
    pub full_name: String,
    pub object: ProfileObject,
    pub updated_at: DateTime<Utc>,
}

// infrastructure db layer
#[derive(sqlx::Type, Debug)]
#[sqlx(type_name = "profile_object", rename_all = "lowercase")]
pub enum ProfileEntityObject {
    Profile,
}

#[derive(Debug)]
pub struct ProfileEntity {
    pub id: String,
    pub full_name: String,
    pub object: ProfileEntityObject,
    pub updated_at: DateTime<Utc>,
}
```

## Spliting Errors

An action can lead to various situations, so the logic behind this is to handle each situation with specific errors to facilitate quick error identification. For example, with the profile functionality, there are different scenarios such as creating a profile, finding a profile, etc.

```rust
#[derive(thiserror::Error, Debug)]
pub enum CreateProfileError {
    #[error("Repository Unavailable.")]
    RepositoryUnavailable,
    #[error("Duplicate Profile Id.")]
    DuplicateProfileId,
    #[error(transparent)]
    Other(anyhow::Error),
}

#[derive(thiserror::Error, Debug)]
pub enum GetProfileError {
    #[error("Repository Unavailable.")]
    RepositoryUnavailable,
    #[error("Profile not found.")]
    NotFound(anyhow::Error),
    #[error(transparent)]
    Other(anyhow::Error),
}
```

I repeat `RepositoryUnavailable`, but that's fine; repetition is good. Having `GetProfileError::RepositoryUnavailable`, for example, allows me to easily identify the operation where the database is not available and research a solution. This is much better than encountering a generic `RepositoryUnavailable` error without knowing when it happened.

## Returning values

I love this habit; it's so simple and helps me to know exactly what data I'm working on. I always, always, always return the values I've added, updated, or deleted. It's up to the caller to ignore the values if they don't need them, but as mentioned, I always return values. Confirmation, audit, feedback mechanisms in the user interface, facilitates testing with expectations, prepare code for future changes. There are numerous benefits to this habit but I've lost count of how many inserts, updates or deletes functions I've encountered that don't return anything 😭

```rust
pub async fn create<'a, E: Executor<'a, Database = Postgres>>(
    executor: E,
    profile_entity: &ProfileEntity,
) -> Result<ProfileEntity, CreateProfileError> {
    let ProfileEntity {
        id,
        full_name,
        ..
    } = profile_entity;
    let profile_entity = sqlx::query_as!(
        ProfileEntity,
        r#"
        insert into profile(id, full_name)
        values ($1, $2)
        returning
            id,
            full_name,
            object,
            updated_at
        "#,
        id,
        full_name,
    )
    .fetch_one(executor)
    .await?;

    Ok(profile_entity)
}
```

For delete operations, I don't need to return everything; the ID alone is sufficient for the caller to perform any necessary operations. The caller can check if the returned ID is correct, clean the cache for this ID, and so on.

```rust
pub async fn delete<'a, E: Executor<'a, Database = Postgres>>(
    executor: E,
    profile_entity: &ProfileEntity,
) -> Result<ProfileEntity, CreateProfileError> {
    let ProfileEntity {
        id,
        ..
    } = profile_entity;
    let profile_entity = sqlx::query_as!(
        ProfileEntity,
        r#"
        delete profile
        where id = $1,
        returning
            id
        "#,
        id,
    )
    .fetch_one(executor)
    .await?;

    Ok(profile_entity)
}
```

## Destructuring

As seen in the post's code, I consistently use "destructuring" for all structs, which makes fields easy to use and reduces boilerplate code.

```rust
let ProfileEntity {
    id,
    full_name,
    ..
} = profile_entity;
```

From the top of my head, these are the habits I always use, and I'm consistent with them, especially in side projects.
