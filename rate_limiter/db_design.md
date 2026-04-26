CREATE TABLE rate_limit_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        VARCHAR(255) NOT NULL,
    tier            VARCHAR(50),
    limit           INT NOT NULL,
    window_seconds  INT NOT NULL,
    algorithm       VARCHAR(50),
    enabled         BOOLEAN DEFAULT true,
    priority        INT DEFAULT 0,
    created_at      TIMESTAMP DEFAULT now() NOT NULL,
    updated_at      TIMESTAMP,
    created_by      VARCHAR(255) NOT NULL,
    updated_by      VARCHAR(255)
);