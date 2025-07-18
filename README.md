[# my-notes]
(https://github.com/Tahanima/playwright-java-test-automation-architecture/tree/main/src)
https://medium.com/@iamfaisalkhatri/playwright-java-tutorial-web-automation-testing-installation-and-setup-545c9c7661c8


-- Test Cases
CREATE TABLE test_cases (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    category TEXT CHECK (category IN ('ui', 'api+db', 'db-only')),
    is_selected BOOLEAN DEFAULT FALSE,
    run_batch_id TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Test Steps
CREATE TABLE test_steps (
    id SERIAL PRIMARY KEY,
    case_id INTEGER REFERENCES test_cases(id) ON DELETE CASCADE,
    step_no INTEGER,
    description TEXT,
    step_type TEXT CHECK (step_type IN ('oracle', 'api', 'elasticsearch', 'ui', 'custom')),
    validation_query TEXT,
    expected_result TEXT,
    ui_action TEXT,
    ui_locator TEXT,
    ui_value TEXT,
    custom_code_ref TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

ALTER TABLE test_steps ADD COLUMN fetch_mode TEXT CHECK (fetch_mode IN ('one', 'all')) DEFAULT 'one';
ALTER TABLE test_steps ADD COLUMN store_as TEXT;


-- Test Results
CREATE TABLE test_results (
    uuid TEXT PRIMARY KEY,
    test_name TEXT,
    step_no INTEGER,
    step_desc TEXT,
    status TEXT CHECK (status IN ('PASSED', 'FAILED')),
    start_time BIGINT,
    stop_time BIGINT,
    environment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Optional: Test Logs for advanced tracing
CREATE TABLE test_logs (
    id SERIAL PRIMARY KEY,
    test_uuid TEXT REFERENCES test_results(uuid),
    log_type TEXT,
    log_content TEXT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);



import json

def assert_db_result(result, expected_result, cursor):
    if not result:
        raise AssertionError("No DB result returned")

    # Normalize expected values to lowercase keys and string values
    expected = {k.lower(): str(v) for k, v in json.loads(expected_result).items()}
    columns = [desc[0].lower() for desc in cursor.description]

    if isinstance(result, list):  # fetchall()
        matched = False
        for row in result:
            row_dict = {k.lower(): str(v) for k, v in zip(columns, row)}
            if all(row_dict.get(k) == expected[k] for k in expected):
                matched = True
                break
        if not matched:
            raise AssertionError(f"Expected match not found in DB result set: {expected}")
    else:  # fetchone()
        row_dict = {k.lower(): str(v) for k, v in zip(columns, result)}
        for k in expected:
            actual = row_dict.get(k)
            assert actual == expected[k], f"Expected {k} = {expected[k]}, but got {actual}"



if store_as:
    columns = [desc[0].lower() for desc in cursor.description]
    if isinstance(result, list):
        if result:
            # Only store the first row by default from fetchall()
            row_dict = dict(zip(columns, result[0]))
        else:
            row_dict = {}
    else:
        row_dict = dict(zip(columns, result)) if result else {}

    for col, val in row_dict.items():
        context[f"{store_as}.{col}"] = str(val)


if step_type == "api" and store_as:
    if response_json and isinstance(response_json, dict):
        def flatten_json(obj, prefix=""):
            flat = {}
            for k, v in obj.items():
                key = f"{prefix}.{k}" if prefix else k
                if isinstance(v, dict):
                    flat.update(flatten_json(v, key))
                else:
                    flat[key] = str(v)
            return flat

        flattened = flatten_json(response_json)
        for key, val in flattened.items():
            context[f"{store_as}.{key}"] = val

-- Test Batches (master table)
CREATE TABLE test_batches (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Mapping Table: test_case_batches (many-to-many relationship)
CREATE TABLE test_case_batches (
    test_case_id INTEGER REFERENCES test_cases(id) ON DELETE CASCADE,
    batch_id INTEGER REFERENCES test_batches(id) ON DELETE CASCADE,
    PRIMARY KEY (test_case_id, batch_id)
);

SELECT tc.id, tc.name
FROM test_cases tc
JOIN test_case_batches cb ON tc.id = cb.test_case_id
JOIN test_batches b ON cb.batch_id = b.id
WHERE b.name = %s AND tc.is_selected = TRUE;

-- Create a batch called 'SMOKE'
INSERT INTO test_batches (name) VALUES ('SMOKE');

-- Create another batch
INSERT INTO test_batches (name) VALUES ('CRITICAL_REGRESSION');

-- Map test cases to SMOKE batch
INSERT INTO test_case_batches (test_case_id, batch_id) VALUES (101, 1);
INSERT INTO test_case_batches (test_case_id, batch_id) VALUES (102, 1);

-- Map test cases to CRITICAL_REGRESSION batch
INSERT INTO test_case_batches (test_case_id, batch_id) VALUES (103, 2);


You can pass the batch name as an environment variable:
ENV=uat BATCH=SMOKE pytest



This framework adopts a database-driven model for managing test cases and test steps. While many teams manually code test cases in files, our approach offers key enterprise advantages:

✅ Why This Is the Right Architecture

🔁 Scalable — Easily manage thousands of test cases (UI/API/DB) without bloated files.

⚙️ Flexible Execution — Filter by batch, category, tag, priority, or owner — no code change required.

📊 Central Visibility — Business, QA, and Developers can all view/edit/manage test cases outside code.

🧩 Reusable Steps — Test actions (e.g. login, form submission) are parameterized and reused across cases.

🧠 Smart Automation — Enables dashboards, analytics, failure tracking, tagging, and priority-based runs.

🛡️ Addressing Concerns

Concern

Response

“Isn’t it more complex than just writing Pytest files?”

Yes at first — but vastly easier to scale and maintain long term.

“What if step logic is too custom for DB?”

Use a hybrid approach — inject references to modular Python logic when needed.


Executive Summary: TestOs One-Pager

TestOs is a scalable, hybrid automation framework designed to handle enterprise-level testing across UI, API, DB, and log layers. It uniquely combines a structured test design model (via PostgreSQL) with real-time business validation (via Oracle), enabling dynamic, data-driven execution.

🚀 Key Highlights:

Modular Framework: Tests are composed of reusable, configurable steps stored in a DB.

Hybrid Capabilities: Supports both DB-managed test execution and traditional file-based tests.

Multi-Layer Validation: UI, API, DB, and ElasticSearch/Kibana log validations all in one flow.

Agentic Test Flow: Dynamically reads test steps and context at runtime — no hardcoded logic.

Autonomous Execution Engine: Dispatches steps based on type, stores results, and injects outputs.

True Business State Validation: Oracle DB is directly queried to assert actual post-transaction outcomes.

Full Traceability: Allure reports + PostgreSQL-based execution logs provide a detailed audit trail.

🎯 Why TestOs?

Avoids code duplication by managing test data and logic separately.

Empowers QA/Dev/BA to collaborate on tests using structured inputs.

Supports 1000+ test cases across multiple domains.

Easily integrates into CI pipelines with batch or tag-based execution.

“Will others be able to write test cases?”

Absolutely. With a UI or admin form, anyone can define steps without touching Python code.

This approach turns your framework into a test platform — not just a script repository.

