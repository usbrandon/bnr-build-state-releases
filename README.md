# bnr-build-state releases

Release downloads for `bnr-build-state`, the Branch & Row build state client. It wraps a dbt
invocation so unchanged models are skipped, using the Branch & Row state service.

This repository publishes release artifacts only. The source is not published here.

## Install

Requires [uv](https://docs.astral.sh/uv/) (or pipx). Replace the version and extra as needed.
The client needs a Branch & Row account and a run token (`BNR_URL`, `BNR_TOKEN`, `BNR_ACCOUNT_ID`).
Without them it runs dbt unchanged.

```
uv tool install "bnr-build-state[snowflake] @ https://github.com/usbrandon/bnr-build-state-releases/releases/download/bnr-build-state-v0.3.11/bnr_build_state-0.3.11-py3-none-any.whl" --with dbt-snowflake
```

Extras: `snowflake`, `postgres`, `duckdb`. dbt itself is not bundled; install the dbt version and
adapter your projects use.

## License

MIT. See [LICENSE](LICENSE).

dbt is a trademark of dbt Labs, LLC. This project is not affiliated with or endorsed by dbt Labs.
It works with dbt.
