# DutyFlow

**DutyFlow** is a web application for automatically generating fair duty schedules for organizations that run continuous services — institutions or companies where someone must be on duty every single day (front desk, on-call support, security shifts, weekly on-call rotations, and so on).

Each member can mark the days they would prefer not to work, and a **genetic algorithm** builds a schedule that covers every day of the period while respecting rest periods, avoiding overlaps between services, and distributing sensitive days (holidays, weekends) fairly based on each person's history.

---

## Table of Contents

- [Key Features](#key-features)
- [Example Use Cases](#example-use-cases)
- [Tech Stack](#tech-stack)
- [Core Concepts](#core-concepts)
- [Data Model](#data-model)
- [Scheduling Rules](#scheduling-rules)
- [The Genetic Algorithm](#the-genetic-algorithm)
- [User Roles](#user-roles)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Management Commands](#management-commands)
- [Project Structure](#project-structure)
- [Theming](#theming)
- [Security](#security)
- [Roadmap](#roadmap)
- [License](#license)

---

## Key Features

- **Automatic schedule generation** for any number of services, using a genetic algorithm.
- **Off-Duty (OD) preferences** — members mark the days in the upcoming period when they do not want to be assigned, with an optional reason.
- **Sensitive day priorities** — holidays and weekends are ranked (Priority 1–4) and assigned fairly, independently per priority level.
- **History-aware fairness** — members who have already performed more duties get their preferences honored first.
- **Cross-service conflict detection** — a person is never scheduled on two services on the same day or during the mandatory rest period after a duty.
- **Full coverage guarantee** — services are continuous, so every day of every period always has someone assigned.
- **Transparent fallbacks** — when a preference cannot be honored, the assigned person receives a descriptive notification explaining why.
- **Interactive calendar** with a color legend per service, names on each day, and a popup with contact details.
- **Role-based access** — administrators manage members, services and day priorities; regular members get a read-only view plus their own preferences.
- **Configurable color scheme** with light and dark themes, driven by CSS variables.
- **Responsive design** that works across desktop and mobile resolutions.

## Example Use Cases

| Service | Shift length | Example |
|---|---|---|
| Daily duty officer | 1 day | One person responsible for the building for 24 hours |
| Security / guard shift | 1 day or 12 hours | Day and night shifts at the main entrance |
| Weekly on-call | 7 days | One person handles incidents for a full week |

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.13, Django 6 |
| Database | PostgreSQL |
| Templates | Django templates with reusable components (`{% include %}`) |
| Frontend | Bootstrap 5, custom CSS (CSS variables), vanilla JavaScript |
| Scheduling | Custom genetic algorithm (pure Python) |

---

## Core Concepts

### Services

A **service** is a recurring duty that must be covered every day of the scheduling period. Each service defines:

- its **name** and description;
- the **shift duration** (e.g. 1 day, 12 hours, 7 days);
- the number of **mandatory rest days** after a shift;
- which members are **eligible** to perform it (derived from their role, with optional per-person overrides).

### Off-Duty (OD) Preferences

Before a schedule is generated, each member can mark days in the upcoming period (e.g. the next 30 days) as **OD**, together with a short reason. OD days are treated as preferences, not guarantees — they are honored in order of how many duties each person has already performed.

### Sensitive Days and Priorities

Administrators can mark days of the upcoming month as **sensitive**. Each sensitive day has a priority level:

| Priority | Type | Examples |
|---|---|---|
| **1** | Major holidays | Christmas, Easter |
| **2** | Holidays that create a long weekend | May 1st falling next to a weekend |
| **3** | Regular public holidays | Other national holidays |
| **4** | Weekends | Saturdays and Sundays |

Fairness is computed **separately for each priority level**: having worked many weekends does not "pay" for Christmas, and vice versa.

---

## Data Model

```
User (extends Django's AbstractUser)
 ├── username, password, email
 ├── first_name, last_name
 ├── work_phone, personal_phone
 ├── role / position
 └── eligible services (M2M override)

Service
 ├── name, description
 ├── shift_duration
 ├── mandatory_rest_days
 ├── eligible roles (M2M)
 └── is_active                (services are deactivated, never deleted)

DayType
 ├── name                     (e.g. Christmas, Easter, Saturday)
 └── priority                 (1–4)

Day
 ├── date
 └── day_type (FK, nullable)  (null = regular working day)

SchedulePeriod
 ├── start_date, end_date
 └── services covered

ScheduleShift                 (output of the genetic algorithm)
 ├── period (FK)
 ├── service (FK)
 ├── user (FK)
 └── date

UserDayOffPreference
 ├── user (FK)
 ├── date
 └── reason
```

History is preserved: services are deactivated instead of deleted, and members who already have scheduled shifts can only be deactivated, never removed.

---

## Scheduling Rules

The algorithm enforces the following rules, listed from strictest to most flexible.

### Hard constraints (must never be violated)

1. **Full coverage** — every service has exactly one person assigned for every day of the period.
2. **Eligibility** — a person can only be assigned to services they are eligible for.
3. **No overlaps** — a person cannot be on two services on the same day.
4. **Rest periods** — after a shift, a person cannot be assigned to *any* service during that shift's mandatory rest days. This is checked across all services, including shifts already stored from previously generated schedules.

### Soft constraints (honored whenever possible)

5. **OD preferences, ordered by workload** — when not every OD can be honored, preferences are respected in descending order of duties already performed: the person who has done the most duties has their OD honored first.
6. **Sensitive-day fairness** — for each priority level, sensitive days are assigned to the people with the fewest duties *at that priority level*. A person who has already worked two Priority-2 days and marks another one as OD should not get it; instead, it goes to whoever has worked the fewest Priority-2 days. This repeats until counts are balanced.
7. **Even overall workload** — the total number of duties per service is spread as evenly as possible.

### Fallback rules

8. If a day cannot be filled under the rules above, the shift goes to the **eligible person with the fewest duties of that service type so far**.
9. If several people are tied for the fewest duties (e.g. A = 10, B = 9, C = 4, D = 4), the shift is assigned **randomly among the tied people** (C or D).

In both fallback cases, the assigned person receives a **descriptive notification** explaining why they were chosen.

---

## The Genetic Algorithm

Scheduling with many overlapping constraints is a combinatorial optimization problem. Instead of trying every possible combination, DutyFlow uses a **genetic algorithm (GA)**: it starts from a population of candidate schedules and evolves them over many generations, keeping the best ones and combining their good parts.

### 1. Encoding (chromosome)

A candidate schedule is a **chromosome**: a grid with one gene per `(service, day)` slot. The value of each gene is the user assigned to that slot.

```
            Day 1   Day 2   Day 3   ...   Day 30
Service A   [ u7 ]  [ u2 ]  [ u9 ]  ...   [ u4 ]
Service B   [ u3 ]  [ u11]  [ u5 ]  ...   [ u8 ]
Service C   [ u1 ]  [ u1 ]  [ u1 ]  ...   [ u6 ]   <- 7-day shift: one person per block
```

For multi-day services (e.g. 7-day shifts), a gene represents a whole shift block rather than a single day, so a block is always assigned to one person.

Because every gene always holds exactly one user, the **full coverage rule is guaranteed by construction**. Each gene is only ever filled with a user who is eligible for that service, so **eligibility is also guaranteed by construction**.

### 2. Initial population

The first generation is built from a mix of:

- **random schedules** — each slot gets a random eligible user (keeps diversity high);
- **greedy schedules** — slots are filled day by day, choosing the eligible user with the fewest duties who is not on rest and has no OD that day (gives the algorithm good starting points).

### 3. Fitness function

Each chromosome receives a **penalty score**; lower is better. Hard-constraint violations are weighted so heavily that any schedule breaking them is always worse than any schedule that respects them.

```
penalty =  W_HARD  × (overlaps + rest-period violations)
         + W_OD    × Σ ignored OD preferences, weighted by the person's duty count
         + W_P1    × imbalance of Priority-1 days
         + W_P2    × imbalance of Priority-2 days
         + W_P3    × imbalance of Priority-3 days
         + W_P4    × imbalance of Priority-4 days
         + W_LOAD  × imbalance of total duties per service

fitness = 1 / (1 + penalty)
```

Key details:

- **Rest-period violations** are checked against the candidate *and* against shifts already saved in the database for other services, so a new schedule never conflicts with an existing one.
- **OD penalties** grow with the number of duties the person has already performed, which makes the algorithm prefer honoring the preferences of people with the heaviest workload (rule 5).
- **Priority imbalance** is measured per level, using each person's historical count for that level plus the days assigned in the candidate (for example, the variance or max–min spread of counts). Weights decrease from Priority 1 to Priority 4 (`W_P1 > W_P2 > W_P3 > W_P4`), so fairness on major holidays matters most.

### 4. Selection

Parents are chosen with **tournament selection**: a few chromosomes are picked at random and the one with the best fitness wins. This keeps pressure toward good solutions without losing diversity too quickly.

**Elitism** copies the best few schedules unchanged into the next generation, so the best solution found is never lost.

### 5. Crossover

Two parents produce a child by combining their genes:

- **Day-block crossover** — a random range of days is taken from parent A and the rest from parent B. Taking whole days (all services at once) keeps each day internally consistent.
- **Service-row crossover** — whole service rows are taken from one parent or the other.

### 6. Mutation

With a small probability, a child is modified:

- **Reassign** — a slot gets a different eligible user.
- **Swap** — two slots of the same service exchange their users (useful for fixing a single OD or rest conflict without changing duty counts).

### 7. Repair step

After crossover and mutation, a quick **repair pass** fixes obvious hard-constraint violations: if a person is on two services on the same day, or is on rest, the slot is reassigned to another available eligible user. This keeps the population mostly valid and helps the algorithm converge much faster.

### 8. Termination

Evolution stops when one of these conditions is met:

- a maximum number of generations is reached;
- the best fitness has not improved for a set number of generations (stagnation);
- a time limit is reached.

### 9. Post-processing and fallback

The best chromosome is checked one final time:

1. Any slot that still breaks a hard constraint is filled using the **fallback rules** — the eligible person with the fewest duties of that type, with a **random tie-break** among people with equal counts.
2. Every person whose OD preference was not honored, or who was chosen by a fallback, receives a **descriptive notification**.
3. The final schedule is saved as `ScheduleShift` records and shown on the calendar.

### GA parameters

| Parameter | Typical value |
|---|---|
| Population size | 100–200 |
| Max generations | 500–2000 |
| Elite count | 2–5 |
| Tournament size | 3–5 |
| Crossover rate | 0.8 |
| Mutation rate | 0.01–0.05 per gene |
| Stagnation limit | 100 generations |

All parameters can be adjusted in the settings (see [Configuration](#configuration)).

---

## User Roles

| Role | Permissions |
|---|---|
| **Superuser** | Full access to everything, including all departments and the Django admin. |
| **Administrator (staff)** | Manages members, services and day priorities; generates schedules; assigns sensitive-day priorities for the next month. |
| **Member** | Views schedules, services and statistics (read-only); manages their own OD preferences; receives notifications. |

---

## Getting Started

### Prerequisites

- Python 3.13+
- PostgreSQL 15+
- Git

### 1. Clone the repository

```bash
git clone https://github.com/NovacAndreiGabriel/DutyFlow.git
cd DutyFlow
```

### 2. Create and activate a virtual environment

**Windows (PowerShell):**

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```

**Linux / macOS:**

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Create the database

```sql
CREATE DATABASE dutyflow_db;
CREATE USER dutyflow_user WITH PASSWORD 'your-password';
GRANT ALL PRIVILEGES ON DATABASE dutyflow_db TO dutyflow_user;
\c dutyflow_db
GRANT ALL ON SCHEMA public TO dutyflow_user;
```

> **Note:** on PostgreSQL 15+, the `GRANT ALL ON SCHEMA public` line is required — and must be run again every time the database is dropped and recreated.

### 5. Configure environment variables

Create a `.env` file in the project root (see [Configuration](#configuration)).

### 6. Run migrations and seed data

```bash
python manage.py migrate
python manage.py seed_roles
python manage.py seed_services
python manage.py seed_day_types
python manage.py createsuperuser
```

### 7. Start the development server

```bash
python manage.py runserver
```

Open <http://127.0.0.1:8000/> in your browser.

---

## Configuration

Settings are read from environment variables:

```env
# Django
DJANGO_SECRET_KEY=change-me
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1

# Database
DB_NAME=dutyflow_db
DB_USER=dutyflow_user
DB_PASSWORD=your-password
DB_HOST=localhost
DB_PORT=5432

# Genetic algorithm (optional overrides)
GA_POPULATION_SIZE=150
GA_MAX_GENERATIONS=1000
GA_MUTATION_RATE=0.02
GA_STAGNATION_LIMIT=100
```

> Never commit your `.env` file. It is listed in `.gitignore`.

---

## Management Commands

| Command | Description |
|---|---|
| `seed_roles` | Creates the default roles / positions used to derive service eligibility. |
| `seed_services` | Creates example services (daily duty, guard shift, weekly on-call). |
| `seed_day_types` | Creates the sensitive day types and their priorities. |
| `generate_test_users` | Generates fake members for local testing. |

---

## Project Structure

```
DutyFlow/
├── accounts/                  # Users, roles, departments, authentication
│   ├── management/commands/
│   ├── models.py
│   ├── forms.py
│   └── views.py
├── scheduling/                # Services, days, schedules, preferences
│   ├── algorithm/             # Genetic algorithm
│   │   ├── chromosome.py      # Encoding and initial population
│   │   ├── fitness.py         # Penalty / fitness function
│   │   ├── operators.py       # Selection, crossover, mutation, repair
│   │   └── engine.py          # Main evolution loop and fallback
│   ├── management/commands/
│   ├── models.py
│   └── views.py
├── templates/
│   ├── base.html
│   └── components/            # Reusable template components
├── static/
│   ├── css/
│   │   ├── theme.css          # Color scheme (CSS variables)
│   │   ├── layout.css
│   │   └── calendar.css
│   └── js/
│       ├── calendar.js
│       ├── form-validation.js
│       └── theme-toggle.js
├── dutyflow/                  # Project settings and root URLs
├── manage.py
├── requirements.txt
└── README.md
```

---

## Theming

All colors are defined as CSS variables in a single file, `static/css/theme.css`. Changing the values there updates the whole application:

```css
:root {
  --theme-primary: #1f4e79;
  --theme-secondary: #6c757d;
  --theme-accent: #f0a500;
  --theme-background: #f8f9fa;
  --theme-surface: #ffffff;
  --theme-text: #212529;
}

[data-bs-theme="dark"] {
  --theme-primary: #4a90d9;
  --theme-background: #121212;
  --theme-surface: #1e1e1e;
  --theme-text: #e9ecef;
}
```

Light and dark modes are switched with Bootstrap 5's `data-bs-theme` attribute, and every custom component uses these variables so it works in both themes.

---

## Security

DutyFlow is built with production deployment in mind:

- **Input validation** on both the client (JavaScript) and the server (Django forms and model validation).
- **CSRF protection** on every form.
- **Password hashing** with Django's default PBKDF2 hasher and password strength validators.
- **Permission checks** on every view — read-only users cannot reach edit actions, even by URL.
- **ORM-only database access**, preventing SQL injection.
- **Secrets in environment variables**, never in the codebase.
- Production settings: `DEBUG=False`, `SECURE_SSL_REDIRECT`, secure and HTTP-only cookies, HSTS.

---

## Roadmap

- [x] Project setup, PostgreSQL configuration, custom User model
- [x] Database schema and Django admin customization
- [x] Seeding commands, login / logout / home
- [ ] Member, service and department management pages
- [ ] Planning page: sensitive days, OD preferences, interactive calendar
- [ ] Genetic algorithm for schedule generation
- [ ] Notifications for fallback assignments
- [ ] Statistics page

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
