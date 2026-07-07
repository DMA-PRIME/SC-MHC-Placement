# South Carolina Mobile Health Clinic Placement Decision Tool

A decision-support web tool for identifying high-impact deployment locations for Mobile Health Clinics (MHCs) in South Carolina. The tool uses geospatial demand data, candidate community sites, and travel-time thresholds to recommend MHC locations that maximize coverage of a selected target population.

This repository operationalizes the mobile health clinic placement framework described in:

> Tanim SH, White DL, Witrick B, Rennert L. **Optimizing mobile health clinic placement via geospatial modeling.** *Public Health in Practice.* 2026;11:100805. https://doi.org/10.1016/j.puhip.2026.100805

The published study demonstrated that optimized MHC placement can substantially increase the uninsured population reached within practical driving and walking thresholds. This web tool translates that framework into an interactive planning workflow for comparing deployment alternatives, reviewing feasibility, and exporting field-ready site lists.

## Using the hosted web tool

No installation is required to use the hosted version of the tool. Users can open the web app link, select the target variable, county/ZIP code, travel threshold, site types, number of MHCs, and then click **Calculate Optimal Sites**.

Web tool link: [insert link here]

---

## Table of contents

- [Overview](#overview)
- [Key features](#key-features)
- [Optimization method](#optimization-method)
- [Travel-time modes](#travel-time-modes)
- [Data inputs](#data-inputs)
- [Target population variables](#target-population-variables)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the app](#running-the-app)
- [Using the web tool](#using-the-web-tool)
- [Outputs and exports](#outputs-and-exports)
- [Suggested repository structure](#suggested-repository-structure)
- [Methodological notes and limitations](#methodological-notes-and-limitations)
- [Citation](#citation)
- [Acknowledgments](#acknowledgments)
- [License](#license)

---

## Overview

Mobile Health Clinics can reduce geographic and transportation barriers by bringing services directly to communities. However, choosing deployment sites based only on convenience can miss high-need populations or create redundant service areas. This tool supports systematic placement decisions by combining:

- candidate MHC deployment locations,
- census block based demand points,
- user-selected target population,
- drive or walk travel-time thresholds,
- a Maximum Coverage Location Problem (MCLP) optimization model,
- interactive maps and exportable site tables.


The app is currently designed for South Carolina ZIP-code-based planning, but the workflow can be adapted to other geographies.

---

## Key features

### 1. Target-guided county and ZIP planning

Users first choose the target category and target measure that define the population or need to optimize. They then select a South Carolina county and ZIP code, or search for any ZIP code directly. When a county is selected, ZIP options are filtered to that county and ranked by the selected target variable when demand data are available.

### 2. Flexible target population selection

The tool can optimize for multiple need measures, including uninsured population, age-specific uninsured groups, total population, age groups, non-white population, Hispanic population, zero-vehicle households, non-English-speaking households, veterans, workers, and other demographic indicators available in the input data. Although disease burden is currently included as a placeholder, the workflow can support disease-specific burden measures if the corresponding data field is added to the input JSON and mapped in the target-variable settings.

### 3. MCLP-based site selection

The core model is a Maximum Coverage Location Problem. It selects a fixed number of MHC deployment sites that maximize the selected target population covered within the chosen travel-time threshold. For example, if the user selects 3 MHCs and a 5-minute drive threshold, the tool recommends a three-site deployment plan that maximizes covered demand within 5 minutes.

### 4. Coverage and travel-time ranking

Deployment plans are ranked lexicographically:

1. highest covered target population first;
2. lowest weighted average nearest travel time among equal-coverage solutions;

This keeps population coverage as the primary objective while using travel time to break ties between similarly performing plans.

### 5. Ranked and distinct alternative deployment plans

The tool separates **Number of MHCs to deploy** from **Alternative plans to show**. For one MHC, it ranks individual candidate sites. For two or more MHCs, it repeatedly solves the MCLP and applies no-good or diversity constraints to generate practical backup configurations.

By default, backup plans prefer more site-distinct alternatives so decision-makers do not receive several plans that repeat most of the same locations.

### 6. Previous-deployment extension planning

When the selected area already has one or more MHC deployment locations, users can enter those prior sites in Advanced settings. The tool can remove demand already covered by previous deployments and optimize the next plan for newly reachable demand. Previous deployment locations can be entered by selecting one or more candidate sites or by entering custom latitude/longitude coordinates.

### 7. Candidate exclusion and recommended-site review

The app supports both pre-run and post-run exclusion workflows:

- **Pre-run exclusion:** remove known infeasible or unavailable candidate sites before optimization.
- **Post-run review:** inspect recommended sites, mark infeasible or unavailable locations, and rerun the optimizer using the remaining candidates.

### 8. Operational feasibility fields

If the candidate site data include feasibility fields, the app displays and exports them for field verification. Supported optional fields include:

- `feasibility_status`
- `parking`
- `restroom`
- `wifi`
- `ada`
- `permission`

These fields are not required for optimization, but they help translate model-selected sites into real-world deployment decisions.

### 9. Fleet-size scenario analysis

The tool can run exact best-plan coverage analysis for multiple fleet sizes, such as 1 through 5 MHCs. This helps users compare marginal coverage gains and understand the potential benefit of adding another mobile clinic.

### 10. Travel-time and distance modes

The app supports a fast **Manhattan-style distance** approximation by default and optional **OSM road-network routing** when network-based accessibility estimates are needed. Users can run drive or walk scenarios with custom travel-time thresholds.

### 11. Interactive maps and plan comparison outputs

The app display county overview maps, ZIP-level candidate sites, selected MHC locations, covered and uncovered demand points, muted non-selected candidates after optimization, and previous deployment locations when extension planning is active.

### 12. Exportable results

Users can export best-plan site lists, all-plan summary tables, field verification files, and GeoJSON files for reporting, GIS use, or operational follow-up.

---

## Optimization method

The core optimization model is a Maximum Coverage Location Problem (MCLP).

Let:

- `i` index candidate MHC sites,
- `j` index demand points,
- `w_j` be the target population weight at demand point `j`,
- `p` be the number of MHCs to deploy,
- `a_ij = 1` if candidate site `i` covers demand point `j` within the selected travel-time threshold, and `0` otherwise,
- `x_i = 1` if candidate site `i` is selected,
- `y_j = 1` if demand point `j` is covered by at least one selected site.

The primary objective is:

```text
maximize sum_j w_j * y_j
```

Subject to:

```text
sum_i x_i = p

y_j <= sum_i a_ij * x_i    for every demand point j

x_i, y_j in {0, 1}
```

The app first maximizes covered demand. When a valid travel-time matrix is available, a second-stage tie-breaker keeps the same maximum coverage value and minimizes weighted travel time among tied solutions.

For multiple alternative plans, the app repeatedly solves the model while adding constraints that prevent duplicate or overly similar site sets.

---

## Travel-time modes

The app supports two accessibility modes.

### Manhattan-style distance, default

This is the fast default option. It computes projected rectilinear distance between each candidate site and demand point, multiplies by a circuity factor, and converts the result to estimated travel time.

Use this mode for quick screening, demonstrations, or situations where road-network routing is not needed.

### Road-network routing, optional

The optional road-network mode uses OSMnx and NetworkX to build an OpenStreetMap-based routing graph. Travel times are estimated from network distance and speed assumptions. Missing speeds are imputed by road class.

Use this mode when a more realistic network-based coverage estimate is needed. The first run for a new geography may take longer because the app must download and cache the local road network.

Important notes:

- Road-network travel time is not live traffic.
- Time-of-day congestion is not modeled.
- OpenStreetMap coverage and road attributes may vary by location.
- The published paper used ArcGIS Network Analyst and Esri StreetMap Premium; this open-source web implementation uses OSMnx for optional network routing.

---

## Data inputs

The app expects a JSON file referenced by `JSON_PATH` in `config.py`.


### Required elements

The app requires:

- `counties`
- `zip_boundaries` or `zips`
- `candidate_facilities` 
- `demand_points` 
- `latitude` and `longitude` for every candidate site
- `latitude` and `longitude` for every demand point
- at least one demand weight variable used by the target selector

### Coordinate format for boundaries

Boundary coordinates in the JSON are expected as:

```text
[latitude, longitude]
```

The app converts these to the longitude-latitude order used by Shapely.

### Candidate facility fields

Required:

| Field | Description |
|---|---|
| `latitude` | Site latitude |
| `longitude` | Site longitude |

Recommended:

| Field | Description |
|---|---|
| `facility_id` | Stable site identifier |
| `name` | Site name |
| `type` | Site category, such as clinic, school, food mall, faith-based organization, community center |
| `address` | Site address |

Optional feasibility fields:

| Field | Description |
|---|---|
| `feasibility_status` | Feasible, infeasible, unavailable, unknown, etc. |
| `parking` | Parking availability |
| `restroom` | Restroom availability |
| `wifi` | WiFi availability |
| `ada` | ADA accessibility |
| `permission` | Permission or venue approval status |

### Demand point fields

Required:

| Field | Description |
|---|---|
| `latitude` | Demand point latitude |
| `longitude` | Demand point longitude |

At least one target variable column should be present, such as `uninsured_pop` or `tot_pop`.

---

## Target population variables

The tool displays only target variables that are present in the demand point data.

| UI label | JSON column |
|---|---|
| Uninsured Population | `uninsured_pop` |
| Total Population | `tot_pop` |
| Disease burden (placeholder) | `tot_hh` |
| Male Adult Population (20+) | `male_adult` |
| Female Adult Population (20+) | `female_adult` |
| Uninsured Under 19 | `uninsured_under19` |
| Uninsured 20-34 | `uninsured_20_34` |
| Uninsured 35-64 | `uninsured_35_64` |
| Uninsured 65+ | `uninsured_65plus` |
| Population 0-5 | `pop_0_5` |
| Population 0-19 | `pop_0_19` |
| Population 20-34 | `pop_20_34` |
| Population 35-64 | `pop_35_64` |
| Population 65+ | `pop_65plus` |
| Non-White Population | `nonwhite_pop` |
| Hispanic Population | `hispanic_pop` |
| Zero-Vehicle Households | `zero_vehicle_hh` |
| Enrolled in School | `enrolled_school` |
| Non-English at Home | `non_english_home` |
| Worker Population | `worker_pop` |
| Veteran Population | `veteran_pop` |

---



## Using the web tool

1. Choose a **target category** and target measure, such as uninsured population.
2. Select a **county** or search any South Carolina ZIP code. When a county is selected, ZIP choices are filtered and ranked using the selected target variable.
3. Choose candidate **site types** to include.
4. Open **Advanced settings** when needed to:
   - add previous deployment locations,
   - prefer site-distinct backup plans,
   - exclude known infeasible sites,
   - enable fleet size scenario analysis,
   - adjust map display settings.
5. Choose the **number of MHCs to deploy**.
6. Choose the number of **alternative plans to show**.
7. Select travel mode: **drive** or **walk**.
8. Select the maximum travel-time threshold.
9. Optional: enable road-network routing for network-based travel-time estimates.
10. Click **Calculate Optimal Sites**.
11. Review the plan comparison table, plan maps, and site-level metrics.
12. Mark infeasible or unavailable recommended sites and rerun, if needed.
13. Export CSV or GeoJSON outputs for reporting or field verification.

---

## Outputs and exports

### On-screen outputs

The tool displays:

- summary statistics for the selected ZIP code,
- ranked deployment plan comparison table,
- selected-site maps for each plan,
- detailed coverage map for each plan,
- site-level metrics,
- fleet size scenario analysis when enabled.

### Plan comparison table

The plan comparison table includes:

- plan rank,
- number of sites in the plan,
- covered target population,
- coverage percentage,
- average nearest travel time,
- loss compared with best plan,
- selected site names,
- selected site types.

### Site-level metrics

For each selected site, the tool reports:

- **Gross covered demand**: demand the site could reach by itself.
- **Marginal contribution**: coverage lost if that site is removed from the selected plan.

The marginal contribution metric is especially useful for multi-site plans because it identifies which selected sites add unique non-overlapping coverage.

### Downloadable files

The app can export:

| Export | Format | Description |
|---|---|---|
| Best Plan Sites | CSV | Site list for the top-ranked plan |
| All Plan Summary | CSV | Summary table for all ranked plans |
| Field Verification | CSV | All plan sites with field feasibility columns |
| Best Plan Sites | GeoJSON | Selected sites from the best plan for GIS use |

---

---

## Methodological notes and limitations

This tool is intended to support planning decisions, not replace local knowledge or community engagement.

Important limitations:

- The model optimizes spatial accessibility and selected demand weights; it does not guarantee site permission, staffing feasibility, community trust, or expected utilization.
- Candidate site quality depends on the completeness and accuracy of the input dataset.
- Demand estimates based on ACS or disaggregated census data can contain uncertainty.
- Manhattan-style distance is a fast approximation, not a true road-network route.
- Live traffic, time-of-day effects, transit access, seasonal demand, and patient preferences are not currently modeled.
- The app evaluates driving and walking separately.

Recommended use:

- Use the tool to shortlist high-potential deployment sites.
- Review recommendations with local partners.
- Check operational feasibility in the field.
- Use exports for site verification, partner discussion, and deployment planning.
- Update candidate and demand data regularly.

---

## Citation

If you use this tool or the underlying placement framework, please cite:

```text
Tanim SH, White DL, Witrick B, Rennert L. Optimizing mobile health clinic placement via geospatial modeling. Public Health in Practice. 2026;11:100805. doi:10.1016/j.puhip.2026.100805
```

BibTeX:

```bibtex
@article{tanim2026mhcplacement,
  title = {Optimizing mobile health clinic placement via geospatial modeling},
  author = {Tanim, Shakhawat H. and White, David L. and Witrick, Brian and Rennert, Lior},
  journal = {Public Health in Practice},
  volume = {11},
  pages = {100805},
  year = {2026},
  doi = {10.1016/j.puhip.2026.100805}
}
```

---

## Optional local setup for developers

Regular users do not need to install anything. The setup instructions below are only for developers or collaborators who want to run the Streamlit app locally, modify the source code, update the input dataset, or deploy another instance of the tool.

Python 3.10 or 3.11 is recommended.

### Option 1: pip

```bash
git clone <repository-url>
cd <repository-name>
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
python -m pip install --upgrade pip
python -m pip install streamlit pandas numpy geopandas shapely folium streamlit-folium osmnx networkx "pulp[cbc]"
```

### Option 2: conda, recommended on Windows for geospatial dependencies

```bash
conda create -n mhc-placement python=3.11 -y
conda activate mhc-placement
conda install -c conda-forge streamlit pandas numpy geopandas shapely folium osmnx networkx pulp -y
python -m pip install streamlit-folium
```

If the CBC solver is not available through PuLP in your environment, install CBC through conda-forge:

```bash
conda install -c conda-forge coincbc -y
```

---

### Configuration

Create a file named `config.py` in the repository root:

```python
from pathlib import Path

JSON_PATH = Path("data/sc_mhc_input.json")
```

---

### Running the app

```bash
streamlit run app.py
```

The app will open in a browser window. If it does not open automatically, Streamlit will print a local URL in the terminal.


### Suggested repository structure

```text
.
├── app.py
├── config.py              # local configuration; do not commit sensitive paths
├── requirements.txt       # optional dependency list
├── README.md
├── data/
│   └── sc_mhc_input.json  # local data; avoid committing confidential data
├── docs/
│   └── screenshot.png     # optional screenshot for GitHub README
└── outputs/               # optional exported results
```


---

## Acknowledgments

This project builds on collaborative mobile health clinic deployment research with Clemson Rural Health and Prisma Health. The underlying research was supported by the National Library of Medicine, the National Institute on Drug Abuse, the CDC Center for Forecasting and Outbreak Analytics, Gilead Sciences, and the South Carolina Center for Rural and Primary Healthcare, as described in the associated publication.

---

## License

Add the repository license here.

The associated publication is open access under the publisher's stated terms. The code license should be specified separately for this repository.
