# PNRR Open Data → Relational Database

A MySQL database reverse-engineered from the 15 official open datasets on Italy's **PNRR** (*Piano Nazionale di Ripresa e Resilienza*, the national Recovery and Resilience Plan) and the complementary **PNC** plan. It covers missions, components, measures, milestones and targets, projects, funding, tenders, the entities involved and their locations.

Built by the **EAS Team** (Andrea Barone, Emanuele Romano, Simone Settimo) for *Contest n°1* at the Università del Salento (Department of Innovation Engineering), supervised by Prof. A. Longo, academic year 2022/2023.

> [!NOTE]
> The data is a **snapshot from late 2022**: the SQL dump was taken on 3 December 2022, and the most recent implementation dataset is Q2 2022. It is not kept up to date. For current figures, use the official PNRR open data portals.

## What's inside

| Folder / file | Content |
|---|---|
| [`Dati/`](Dati) | The 15 source datasets as semicolon-separated CSV files, each with its official metadata (`*_Metadati.zip`) |
| [`Modelli di Progettazione/`](Modelli%20di%20Progettazione) | Conceptual model (EER diagram) and logical model (relational schema) |
| [`Data Dictionary/`](Data%20Dictionary) | Every column of the database: table, name, description, type, max length, distinct values, source dataset and source table (PDF and Excel) |
| [`Database/script_database.sql`](Database/script_database.sql) | MySQL 8 dump with the full schema **and all the data** |
| [`Report dettagliato.pdf`](Report%20dettagliato.pdf) | The full report (in Italian): datasets, problems found, design choices, pros and cons, queries |

## Source datasets

| # | Dataset | Topic |
|---|---|---|
| 1 | `Mappatura_PNRR_SNSvS_Agenda2030` | PNRR measures mapped to the UN Agenda 2030 goals and the national sustainable-development strategy |
| 2 | `PNRR_MeT_OA_programmazione` | Planned milestones and targets (M&T) for each measure, with EU verification arrangements |
| 3 | `MeT_Attuazione_2021-T4` | Implementation status of the M&T due in Q4 2021 |
| 4 | `MeT_Attuazione_2022-T2` | Implementation status of the M&T due in Q2 2022 |
| 5 | `OpenData_PNRR_QuadroFin_Programmazione` | Financial framework per measure: investment or reform, loan or grant, owning administration, amounts |
| 6 | `TAG_sostegno_clima_digitale` | Climate and digital intervention fields and coefficients per sub-measure (EU Reg. 2021/241, Annex VI) |
| 7 | `AmmTitolari_interoperabilitàBDAP` | Owning administrations and their organisational units, linked to the BDAP and IPA registries |
| 8 | `PNRR_Localizzazione` | Where projects take place (region, province, municipality) |
| 9 | `PNRR_Progetti` | Projects (CUP code) and their funding sources |
| 10 | `PNRR_Soggetti` | Entities involved in projects (planner, implementer, beneficiary) |
| 11 | `PNRR_Gare` | Tenders linked to projects (CIG code) |
| 12 | `PNRR_TAG_Progetti` | Intervention fields and climate/digital coefficients per project |
| 13 | `OpenData_PNC_Obiettivi` | Objectives of the complementary plan (PNC) per programme and investment, with deadlines |
| 14 | `OpenData_PNC_DescrizioneProgrammi` | PNC programmes |
| 15 | `OpenData_PNC_QuadroFinanziario` | PNC financial framework per programme (co-financed with the PNRR or additional) |

## Design

![EER diagram](Modelli%20di%20Progettazione/Diagramma%20EER.png)

*Conceptual model (EER). The relational schema derived from it is in [`Modello relazionale.png`](Modelli%20di%20Progettazione/Modello%20relazionale.png).*

The final database has:

- **26 entities**, including a disjoint specialisation of `TRAGUARDO` into `MILESTONE` and `TARGET`;
- **31 relationships**, 8 of them many-to-many and one derived from a multi-valued attribute;
- **35 tables** in total.

The main hierarchy is **mission → component → measure → sub-measure**. Projects (identified by their CUP code) hang off sub-measures and are linked to entities, funders, tenders, intervention fields and locations. Milestones and targets are linked to the sub-measures that reach them.

### Problems in the source data and how they were handled

The datasets are mostly exports of joins: cartesian products of the tables of the original government database. Turning them back into a normalised schema meant dealing with:

- **non-unique keys** (for example among the owning administrations), solved with progressive surrogate IDs;
- **formats MySQL Workbench couldn't import**: dates went from `dd/mm/yyyy` to `yyyy-mm-dd`, and descriptions contained invalid characters;
- **many NULLs and repeated values**, caused by the denormalised source.

For example, `PNRR_Progetti` has one column per funding source (EU, region, province, municipality, other public, private, PNRR, PNC, still to be found), but in practice only the PNRR column and the total were filled. The sources became a `finanziatore` table with a many-to-many `finanziato_da` relationship to `progetto`, and the total became a derived attribute.

The result keeps **100% of the information** in the 15 datasets while storing as few NULLs and duplicates as possible. One trade-off is that secondary tables (regions, provinces, municipalities, Agenda 2030 goals, indicators) only contain the values that appear in the datasets. For example, 14 of the 17 Agenda 2030 goals are present, because no sub-measure refers to goals 2, 15 and 17.

## Loading the database

The dump comes from MySQL 8.0.31 and doesn't create a database by itself, so create one first:

```bash
mysql -u root -p -e "CREATE DATABASE pnrr CHARACTER SET utf8mb4;"
mysql -u root -p pnrr < Database/script_database.sql
```

## Example queries

The report presents these queries:

1. Number of tuples per table and per foreign-key join
2. Number of NULL values per attribute
3. Number of distinct values per attribute
4. Regions with at least one project, and how many projects each has
5. Maximum number of projects planned by a single entity
6. Intervention fields not linked to any sub-measure
7. Provinces with projects in more than 20 different municipalities
8. Targets reached by each project, when they have a description
9. Sub-measures whose percentage target equals the highest percentage target

For example, query 4:

```sql
SELECT nome_regione, COUNT(*) AS N_progetti
FROM progetto JOIN regione ON progetto.ID_regione = regione.iD_regione
GROUP BY progetto.ID_regione;
```

## Team

**EAS Team**: Andrea Barone, Emanuele Romano, Simone Settimo.
