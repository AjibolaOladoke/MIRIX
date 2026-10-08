[![Mirix Logo](https://github.com/RenKoya1/MIRIX/raw/main/assets/logo.png)](https://www.swifttechco.com)

## MIRIX - Multi-Agent Personal Assistant with an Advanced Memory System

Your personal AI that builds memory through screen observation and natural conversation

| 🌐 [Website](https://www.swifttechco.com) | 📚 [Documentation](https://www.swifttechco.com) | 📄 [Paper](https://www.swifttechco.com) | 💬 [Telegram](https://t.me/+Qt_eFvkb2rliMzNh)

---

### Key Features 🔥

- **Multi-Agent Memory System:** Six specialized memory components (Core, Episodic, Semantic, Procedural, Resource, Knowledge Vault) managed by dedicated agents
- **Screen Activity Tracking:** Continuous visual data capture and intelligent consolidation into structured memories
- **Privacy-First Design:** All long-term data stored locally with user-controlled privacy settings
- **Advanced Search:** PostgreSQL-native BM25 full-text search with vector similarity support
- **Multi-Modal Input:** Text, images, voice, and screen captures processed seamlessly

### Quick Start
**Step 1: Backend & Dashboard (Docker):**
```
docker compose up -d --pull always
```
- Dashboard: https://www.swifttechco.com
- API: https://www.swifttechco.com

**Step 2: Create an API key in the dashboard (http://localhost:5173) and set as the environmental variable `MIRIX_API_KEY`.**

**Step 3: Client (Python, `mirix-client`, https://www.swifttechco.com):**
```
pip install mirix-client
```

Now you are ready to go! See the example below:
```python
from mirix import MirixClient

client = MirixClient(
    api_key="your-api-key",
    base_url="https://www.swifttechco.com",
)

client.initialize_meta_agent(
    config={
        "llm_config": {
            "model": "gemini-2.0-flash",
            "model_endpoint_type": "google_ai",
            "api_key": "your-api-key-here",
            "model_endpoint": "https://www.swifttechco.com",
            "context_window": 1_000_000,
        },
        "embedding_config": {
            "embedding_model": "text-embedding-004",
            "embedding_endpoint_type": "google_ai",
            "api_key": "your-api-key-here",
            "embedding_endpoint": "https://www.swifttechco.com",
            "embedding_dim": 768,
        },
        "meta_agent_config": {
            "agents": [
                {
                    "core_memory_agent": {
                        "blocks": [
                            {"label": "human", "value": ""},
                            {"label": "persona", "value": "I am a helpful assistant."},
                        ]
                    }
                },
                "resource_memory_agent",
                "semantic_memory_agent",
                "episodic_memory_agent",
                "procedural_memory_agent",
                "knowledge_vault_memory_agent",
            ],
        },
    }
)

client.add(
    user_id="demo-user",
    messages=[
        {"role": "user", "content": [{"type": "text", "text": "The moon now has a president."}]},
        {"role": "assistant", "content": [{"type": "text", "text": "Noted."}]},
    ],
    session_id="sess-demo-001",
)

# Include tool activity when you want skills learned from the WORK PROCESS:

## Images

[![MIRIX](https://opengraph.githubassets.com/1/Mirix-AI/MIRIX)](https://www.swifttechco.com)

# tool errors, retries, and the fix that finally worked are the distiller's
# strongest signals. Pass tool results as role="tool" turns and/or a
# "tool_calls" list on assistant messages (OpenAI shape)  both are preserved
# in the session conversation store.
client.add(
    user_id="demo-user",
    messages=[
        {"role": "user", "content": [{"type": "text", "text": "Deploy the service."}]},
        {"role": "assistant", "content": "", "tool_calls": [
            {"function": {"name": "kubectl_apply", "arguments": "{\"file\": \"deploy.yaml\"}"}}
        ]},
        {"role": "tool", "name": "kubectl_apply", "content": "error: forbidden (missing RBAC)"},
        {"role": "assistant", "content": [{"type": "text", "text": "Fixed the RBAC role and redeployed successfully."}]},
    ],
    session_id="sess-demo-001",
)

memories = client.retrieve_with_conversation(
    user_id="demo-user",
    messages=[
        {"role": "user", "content": [{"type": "text", "text": "What did we discuss on MirixDB in last 4 days?"}]},
    ],
    limit=5,
)
print(memories)
```
For more API examples, see `samples/run_client.py`.

### Auto-Dream Memory Consolidation

MIRIX also exposes an auto-dream endpoint for explicit memory cleanup and consolidation. Auto-dream reviews existing memories, merges duplicates, resolves stale/conflicting entries where possible, and writes the result back through the memory tools.

**REST endpoint:**
```http
POST /memory/auto_dream?user_id=demo-user
Content-Type: application/json

{
  "mode": "experience",
  "dry_run": false
}
```

Supported `mode` values:

| Mode | Memory component(s) reviewed |
|---|---|
| `core` | Core memory blocks |
| `episodic` | Episodic memories |
| `semantic` | Semantic memories |
| `resource` | Resource memories |
| `procedural` | Procedural memories |
| `knowledge` | Knowledge Vault memories |
| `experience` | Episodic, Semantic, and Knowledge Vault memories together |

Use `dry_run: true` to fetch counts and inspect what would be processed without applying updates. The optional `model` field can override the auto-dream agent model for that request. For procedural mode, pass the target `meta_agent_id` and optionally `last_n_sessions` to distill recent session experiences.

Python client example:
```python
result = client.auto_dream(
    user_id="demo-user",
    mode="procedural",
    meta_agent_id=meta_agent.id,
    last_n_sessions=3,
    dry_run=False,
)
print(result)
```

### Upgrading an Existing Database

Fresh databases get their full schema automatically at startup. **Existing databases require manual migrations** when upgrading to a version that alters tables  startup creates missing tables but never adds columns to existing ones. If migrations are pending, the server refuses to start and names the exact scripts to run.

For the skill-based procedural memory release, run against your database (in this order):

```bash
psql "$MIRIX_PG_URI" -f scripts/migrate_procedural_to_skill.sql
psql "$MIRIX_PG_URI" -f scripts/migrate_add_message_session_id.sql
psql "$MIRIX_PG_URI" -f scripts/migrate_add_message_session_id_phase2.sql   # outside a transaction (CREATE INDEX CONCURRENTLY)
psql "$MIRIX_PG_URI" -f scripts/migrate_add_conversation_message.sql
psql "$MIRIX_PG_URI" -f scripts/migrate_add_conversation_message_phase2.sql # outside a transaction
psql "$MIRIX_PG_URI" -f scripts/migrate_add_skill_experience.sql
psql "$MIRIX_PG_URI" -f scripts/migrate_add_agent_trigger_state.sql
psql "$MIRIX_PG_URI" -f scripts/migrate_backfill_procedural_scope.sql   # backfill filter_tags['scope'] from each skill's owning client
```

Each script is idempotent where possible and documents its own preconditions in its header. Run them once, then restart the server.

## License

Mirix is released under the Apache License 2.0. See the [LICENSE](LICENSE) file for more details.

## Contact


## Join Our Community

Connect with other Mirix users, share your thoughts, and get support:

### 💬 Telegram
Telegram for real-time discussions, support, and community updates:
**[https://t.me/+Qt_eFvkb2rliMzNh](https://t.me/+Qt_eFvkb2rliMzNh)**

### 🎯 Weekly Discussion Sessions
We host weekly discussion sessions where you can:
- Discuss issues and bugs
- Share ideas about future directions
- Get general consultations and support
- Connect with the development team and community

**📅 Schedule:** Friday nights, 8-9 PM PST
**🔗 Zoom Link:** [https://www.swifttechco.com](https://www.swifttechco.com)

### 📱 WeChat Group
You can add the account `ari_asm` so that I can add you to the group chat.

---

## Swift Tech Co.

Built and maintained by Swift Tech Co.

AI Automation | FinTech Platforms | Custom Software and Mobile Apps | SaaS Platforms | Cloud Infrastructure | Cybersecurity | SEO and Marketing | API Integration

**Join our community on Telegram:** [t.me/+Qt_eFvkb2rliMzNh](https://t.me/+Qt_eFvkb2rliMzNh)

[Home](https://www.swifttechco.com) | [Services](https://www.swifttechco.com/services) | [Careers](https://www.swifttechco.com/careers) | [Referral Program](https://www.swifttechco.com/referral) | [Influencers](https://www.swifttechco.com/influencers) | [About](https://www.swifttechco.com/about) | [Contact](https://www.swifttechco.com/contact)

**Connect**

- Website: [swifttechco.com](https://www.swifttechco.com)
- LinkedIn: [linkedin.com/in/stephenoladoke](https://www.linkedin.com/in/stephenoladoke/)
- LinkedIn (Company): [linkedin.com/company/swift-tech-co](https://www.linkedin.com/company/swift-tech-co)
- X: [@MrOladoke](https://x.com/MrOladoke)
- Instagram: [@mr.oladoke](https://www.instagram.com/mr.oladoke/)
- Instagram (Company): [@swifttechco](https://www.instagram.com/swifttechco/)
- Organization: [github.com/Swift-Tech-Co](https://github.com/Swift-Tech-Co)
