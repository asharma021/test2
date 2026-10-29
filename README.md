uv run --no-sync extract-load-postgres --tag iw

uv run --no-sync python -c "import psycopg; import yaml; print('Dependencies available')"
