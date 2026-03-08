# AutoSOC: Questions and Answers

## Q&A

### Why Google ADK instead of Vertex AI Agent Builder?

ADK is Google's code-first agent framework built on top of the same Vertex AI infrastructure that Vertex AI Agent Builder uses. The difference is control. Vertex AI Agent Builder is a managed, low-code product where Google handles the agent loop, tool registration, and session management. ADK exposes all of that as Python code that you write, own, and version-control. For a security system, that distinction matters: every decision the agent makes is visible in code, testable in isolation, and auditable in the ADK trace UI. Vertex AI Agent Builder abstracts those decisions away. ADK also deploys the same way as any other Python application on Cloud Run, whereas Vertex AI Agent Builder is locked to the managed console. For production security infrastructure, code-first control over the agent behavior is the correct tradeoff.

### Why Pub/Sub for inter-agent communication instead of direct function calls?

Pub/Sub decouples agents so that each one is independently deployable, scalable, and restartable without affecting the others. Every message between agents is a durable, auditable event. This means an investigation can survive an agent restart mid-pipeline, and a complete communication trace is available in Cloud Monitoring without any additional instrumentation. Direct function calls would create tight coupling and eliminate auditability, which is exactly the wrong tradeoff for a security system.

### How does the human-in-the-loop gate work?

The Remediation Agent checks the triage severity score against a configurable threshold (`AUTO_REMEDIATE_MAX_SCORE = 6` in `shared/config.py`). Scores at or below the threshold trigger automatic execution of the remediation action (for example, setting `public_access_prevention = "enforced"` on a GCS bucket). Scores above the threshold send a structured message to a Slack webhook containing the investigation ID, severity score, resource name, and recommended action. The human approver reviews the evidence in Firestore and the Looker Studio dashboard before approving. This threshold is configurable per deployment to match an organization's risk tolerance.

### Why Firestore for investigation state instead of BigQuery?

Firestore is a document database optimized for low-latency reads and writes on individual records. An investigation is a state machine with frequent partial updates as each agent completes its step. Writing an agent's output to Firestore and having the next agent read it takes under 100ms. BigQuery is a columnar analytics warehouse optimized for large-scale aggregate queries, not for individual document updates. The correct pattern is Firestore for operational investigation state and BigQuery for historical analytics across all investigations.

### How does the severity scoring work in the Triage Agent?

The base score is 5. Points are added based on SCC severity (+3 for CRITICAL, +2 for HIGH, +1 for MEDIUM), principal role sensitivity (+2 if owner or editor, +1 if BigQuery access, +1 if Storage access), and behavioral deviation (+1 if the score already exceeds 7, indicating unusual activity). The score is capped at 10. This means a CRITICAL finding by a service account with owner-level access scores 10 and always triggers human approval regardless of the auto-remediation threshold.

### What happens if an agent fails mid-investigation?

The investigation state in Firestore records which agents have completed via an `agents_completed` array. If an agent fails, the investigation document retains its last known state. The Orchestrator can be re-triggered with the existing investigation ID, which will resume from the last completed step rather than starting over. Full re-runs are also supported by submitting the same finding ID with a new investigation ID.

### Why does the Reporting Agent use Vertex AI while the ADK Orchestrator uses the Gemini API key?

The ADK framework running the Orchestrator uses a Gemini API key (from AI Studio) for the conversational agent loop. The Reporting Agent runs as a standalone Python function outside ADK and uses Vertex AI (Application Default Credentials, project-billed) for NL summary generation. This is intentional: the Reporting Agent is a production component that should be billed to the GCP project and use enterprise-grade credentials, whereas the ADK orchestration layer is the development interface.

### How would this deploy to production?

Each agent would be packaged as a Cloud Run service triggered by its corresponding Pub/Sub subscription. The ADK Orchestrator would run as a Cloud Run service with a public-facing endpoint protected by IAP or Cloud Endpoints. The Gemini API key would be replaced by Vertex AI Application Default Credentials throughout. SCC would be configured to export findings to the `scc-findings-raw` Pub/Sub topic via a notification config, creating a fully automated pipeline from real finding to resolved investigation.

### What are the cost implications of running AutoSOC continuously?

The dominant cost is Gemini API calls. Each investigation involves approximately 7 Gemini requests: one per agent plus the ADK orchestration turns. At current Vertex AI pricing for Gemini 2.0 Flash, a full investigation costs under $0.01 in model inference. Pub/Sub, Firestore, BigQuery, and Cloud Logging costs are negligible at the volumes typical of a single GCP project's SCC finding rate (typically 10-100 findings per day for a moderately active project). A realistic monthly cost for continuous operation on a single project is under $50.

### What security controls are on the AutoSOC system itself?

The service account `autosoc-sa` follows least-privilege: it has viewer access to SCC and logging, editor access to Pub/Sub and Firestore, and write access to BigQuery and GCS. It does not have project-level IAM admin rights. All Pub/Sub topics are internal to the project and not publicly accessible. The GCS evidence bucket uses uniform bucket-level access with public access prevention enforced. The ADK web UI is local-only and not exposed to the internet.
