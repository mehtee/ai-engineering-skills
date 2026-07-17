# Settings (`pydantic-settings`)

Package: `pydantic-settings` (separate from core `pydantic`).

## Basic BaseSettings

```python
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_prefix="APP_",
        extra="ignore",
        case_sensitive=False,
    )

    debug: bool = False
    database_url: str
    api_key: SecretStr
    max_connections: int = Field(default=10, ge=1, le=1000)


settings = Settings()  # reads env + .env
```

Env names: `APP_DEBUG`, `APP_DATABASE_URL`, `APP_API_KEY`, …

## Nested settings

```python
class DbSettings(BaseSettings):
    host: str = "localhost"
    port: int = 5432


class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(env_nested_delimiter="__")

    db: DbSettings = Field(default_factory=DbSettings)
    # APP_DB__HOST, APP_DB__PORT
```

## Sources priority (default mental model)

Highest → lowest typically: init kwargs > environment > dotenv file > secrets dir > defaults.  
Customize with `settings_customise_sources` when order must change.

```python
from pydantic_settings import PydanticBaseSettingsSource


class Settings(BaseSettings):
    ...

    @classmethod
    def settings_customise_sources(
        cls,
        settings_cls,
        init_settings: PydanticBaseSettingsSource,
        env_settings: PydanticBaseSettingsSource,
        dotenv_settings: PydanticBaseSettingsSource,
        file_secret_settings: PydanticBaseSettingsSource,
    ):
        return init_settings, env_settings, dotenv_settings, file_secret_settings
```

## Secrets directory

```python
model_config = SettingsConfigDict(secrets_dir="/run/secrets")
```

Docker/K8s style: file name = field name, content = value.

## CLI (optional)

`pydantic-settings` can expose CLI parsing in recent versions (`cli_parse_args`, etc.). Prefer explicit CLI libraries (typer/click) for complex CLIs; use settings for env-backed config.

## Best practices

1. **Single settings object** per process; instantiate once at startup.
2. **`SecretStr`** for keys/passwords; never print `model_dump()` in logs without redaction.
3. Fail fast: required env vars with no default → crash on boot, not mid-request.
4. Do not commit real `.env` secrets; provide `.env.example` with placeholders.
5. Use `env_prefix` to avoid collisions in shared hosts.
6. For tests: `Settings(database_url="sqlite://", _env_file=None)` or monkeypatch env.
7. Prefer `extra="ignore"` on settings so unknown env noise does not break boots; document required vars.
8. Typed URLs: `PostgresDsn`, `AnyUrl` from pydantic when helpful.

## FastAPI wiring

```python
from functools import lru_cache


@lru_cache
def get_settings() -> Settings:
    return Settings()


# dependency
def endpoint(settings: Settings = Depends(get_settings)):
    ...
```

`lru_cache` ensures one load; clear cache in tests when env changes.
