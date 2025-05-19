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



def assert_db_result(result, expected_result, cursor):
    import json
    if not result:
        raise AssertionError("No DB result returned")
    expected = json.loads(expected_result)
    columns = [desc[0].lower() for desc in cursor.description]

    if isinstance(result, list):  # fetchall()
        matched = False
        for row in result:
            row_dict = dict(zip(columns, row))
            if all(str(row_dict.get(k)) == str(v) for k, v in expected.items()):
                matched = True
                break
        if not matched:
            raise AssertionError(f"Expected match not found in DB result set: {expected}")
    else:  # fetchone()
        row_dict = dict(zip(columns, result))
        for k, v in expected.items():
            actual = str(row_dict.get(k))
            assert actual == str(v), f"Expected {k} = {v}, but got {actual}"
