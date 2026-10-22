# oci-metrics-report agent guide

OCI Monitoring MQL metrics query and report generator with a Flask dashboard and CLI.

## Commands
- Install: `pip install -r requirements.txt` (requirements.txt).
- Test: no test suite is present.
- Lint: no lint command is defined in the repository.
- Run web app: `python app.py` or `./run.sh`.
- Generate CLI report: `python generate_report.py -c <COMPARTMENT_OCID> -n <NAMESPACE> -m <METRIC>`.
- CloudShell wrapper: `./cloudshell_report.sh <COMPARTMENT_OCID> <NAMESPACE> <METRIC>`.

## Layout
- Flask API and OCI clients -> `app.py`
- CLI report generation -> `generate_report.py`
- Startup and dependency setup -> `run.sh`
- CloudShell CLI wrapper -> `cloudshell_report.sh`
- Browser dashboard -> `static/index.html`
- Screenshots -> `images/`
- User and architecture documentation -> `README.md`, `ARCHITECTURE.md`

## Boundaries
- OCI config authentication defaults to `~/.oci/config`, profile `DEFAULT`; `OCI_CONFIG_FILE`, `OCI_CONFIG_PROFILE`, and `OCI_REGION` can override settings.
- `.gitignore` excludes local `.oci/`, `.env` files, virtual environments, and generated report files; keep credentials and real OCIDs out of commits.

## On error
Search KB.md (and ~/dev/OCI-DEMO/KB.md for OCI issues) before debugging; add an entry after a new fix. Do not read KB.md at session start.

## Done
- `python -m py_compile app.py generate_report.py`

<!-- BEGIN oci4cca-ecosystem-matrix -->
Shared OCI4CCA ownership/interconnect contract: [docs/OCI4CCA_ECOSYSTEM.md](docs/OCI4CCA_ECOSYSTEM.md), generated from OCI-DEMO `config/oci4cca_ecosystem.json`. Read it only for cross-project work.
<!-- END oci4cca-ecosystem-matrix -->
