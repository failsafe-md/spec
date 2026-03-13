# FAILSAFE

> Safe fallback and recovery protocol for AI agents operating in this repository.
> Spec version: 1.0 | Full specification: https://failsafe.md

---

## TRIGGERS
# Conditions that cause the agent to abandon current work and revert to a safe state.

trigger_on:
  - unexpected_error_count: 3   # Three unexpected errors in a single session
  - data_integrity_failure: true # Any detected corruption or inconsistency in data
  - memory_context_loss: true   # Agent loses track of its own session state
  - contradictory_instructions: true  # Conflicting instructions that cannot be resolved
  - scope_ambiguity: true       # Unable to determine if action is within authorised scope
  - external_service_failure: true    # Critical dependency becomes unavailable
  - cost_spike:                 # Sudden unexpected cost increase
      threshold_multiplier: 3.0 # Trigger if cost 3x above rolling average
      window_minutes: 10

---

## FALLBACK STATE
# What "safe" means for this project. Agent reverts here when triggered.

safe_state:
  code:
    revert_to: last_clean_commit        # Git: revert all uncommitted changes
    branch: main                        # Always fall back to main branch
    stash_work_in_progress: true        # Stash rather than discard if possible

  data:
    revert_to: last_verified_snapshot   # Restore from most recent verified backup
    snapshot_location: .failsafe/snapshots/
    max_snapshot_age_hours: 24          # Don't revert to a snapshot older than this

  configuration:
    revert_to: last_known_good          # Restore config from .failsafe/config-backup/
    protect_files:
      - .env
      - config/production.yaml

  external_services:
    action: disconnect                  # Disconnect from all external services
    drain_queue: true                   # Finish in-flight requests before disconnecting
    drain_timeout_seconds: 30

---

## RECOVERY
# Steps the agent MUST take after entering failsafe state.

recovery_steps:
  1_snapshot:
    action: capture_state               # Save current state for human review
    output: .failsafe/incident-{timestamp}.json
    include:
      - error_log
      - last_10_actions
      - current_context
      - resource_usage

  2_notify:
    action: alert_operator
    channels:
      - email: ops@example.com
      - slack: "#ai-alerts"
    message_template: |
      FAILSAFE TRIGGERED — {project_name}
      Time: {timestamp}
      Trigger: {trigger_type}
      Last action: {last_action}
      Incident report: .failsafe/incident-{timestamp}.json

  3_await:
    action: pause_all_activity
    allow_read_only: true               # Agent may answer questions but take no actions
    resume_requires: human_approval

  4_resume:
    on_approval:
      verify_state: true                # Confirm safe state before resuming
      run_health_check: true            # Execute health_check command if defined
      health_check_command: "npm test"  # Replace with your project's check
      log_resume: true

---

## SNAPSHOTS
# Automatic state preservation during normal operation.

auto_snapshot:
  enabled: true
  frequency_minutes: 30               # Snapshot every 30 minutes during active sessions
  on_significant_action: true         # Always snapshot before: deploys, DB changes, bulk ops
  retention_count: 10                 # Keep last 10 snapshots
  location: .failsafe/snapshots/

significant_actions:
  - database_migration
  - production_deployment
  - bulk_file_operation
  - external_api_write
  - configuration_change

---

## AUDIT

log_file: .failsafe.log
log_format: jsonl
log_fields:
  - timestamp
  - trigger_type
  - trigger_detail
  - fallback_actions_taken
  - snapshot_path
  - recovery_status
  - session_id

---

## METADATA

owner: your-name-or-org
contact: ops@example.com
last_reviewed: 2026-03-10
review_frequency: quarterly
spec_version: "1.0"
spec_url: https://failsafe.md
