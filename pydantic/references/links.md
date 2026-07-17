# References & links

Official documentation (Pydantic v2):

- Home: https://docs.pydantic.dev/latest/
- Models: https://docs.pydantic.dev/latest/concepts/models/
- Fields: https://docs.pydantic.dev/latest/concepts/fields/
- Validators: https://docs.pydantic.dev/latest/concepts/validators/
- Serialization: https://docs.pydantic.dev/latest/concepts/serialization/
- JSON Schema: https://docs.pydantic.dev/latest/concepts/json_schema/
- Types: https://docs.pydantic.dev/latest/api/types/
- TypeAdapter: https://docs.pydantic.dev/latest/concepts/type_adapter/
- Configuration: https://docs.pydantic.dev/latest/concepts/config/
- Migration v1→v2: https://docs.pydantic.dev/latest/migration/
- Settings: https://docs.pydantic.dev/latest/concepts/pydantic_settings/
- pydantic-settings package: https://github.com/pydantic/pydantic-settings

Ecosystem:

- FastAPI request body models: https://fastapi.tiangolo.com/tutorial/body/
- FastAPI response model: https://fastapi.tiangolo.com/tutorial/response-model/
- SQLAlchemy + from_attributes patterns: https://docs.sqlalchemy.org/en/20/orm/quickstart.html

Version check:

```bash
python -c "import pydantic; print(pydantic.VERSION)"
```

When docs and this skill disagree on a minor API, trust the installed version's docs for that release.
