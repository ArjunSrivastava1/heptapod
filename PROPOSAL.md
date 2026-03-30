# GSoC 2026 — HEPTAPOD Structural Improvements Proposal

**Applicant:** Arjun Srivastava  
**GitHub:** ArjunSrivastava1  
**Project:** Agentic AI for High Energy Physics Analyses at the CMS Detector  
**Branch:** gsoc2026-arjun-srivastava

---

## 1. Current Structure Analysis

After reviewing the repository, the current structure is:
```
heptapod/
├── config.py              # Root-level config
├── test_runner.py         # Root-level test runner
├── llm/                   # LLM utilities
├── prompts/               # System prompts
├── tools/                 # Physics tools
│   ├── analysis/
│   ├── feynrules/
│   ├── inspire/
│   ├── mg5/
│   ├── pdg/
│   ├── pythia/
│   ├── sherpa/
│   └── units/
└── examples/              # Mixed: MCP servers, demos, shared utils, workflows
    ├── hep_bsm_sandbox/
    ├── mcp/               # MCP servers live here
    ├── orchestral/
    ├── setup/
    ├── shared/            # Shared utilities inside examples — problematic
    └── workflows/
```

### Identified Issues

**Issue 1 - examples/ conflation:**
Shared utilities (`shared/llm_utils.py`, `shared/tool_logger.py`) 
live inside `examples/` but are used across the codebase. 
Utilities should not depend on examples existing.

**Issue 2 - MCP as an afterthought:**
The MCP server (`examples/mcp/`) is a first-class integration 
mechanism but is buried inside examples. As AI tooling 
increasingly adopts MCP, this should be a top-level module.

**Issue 3 - No evaluation layer:**
There is no mechanism to measure agent output quality — 
faithfulness, tool call accuracy, or answer relevance. 
Without evaluation, improvements cannot be measured.

**Issue 4 - No memory/context persistence:**
Each agent run starts fresh. There is no mechanism for 
agents to reference previous analyses or build institutional 
knowledge over time.

**Issue 5 - Scattered test infrastructure:**
Test files live inside each tool directory with no consistent 
pattern. Some tools have `tests/` subdirectories, others have 
test files at the same level as source files.

**Issue 6 - No multi-agent coordination:**
The framework supports single-agent workflows only. 
Complex HEP analyses could benefit from specialized 
sub-agents (simulation agent, analysis agent, literature agent) 
coordinated by a router agent.

---

## 2. Proposed Folder Structure
```
heptapod/
├── core/                      # Framework internals
│   ├── __init__.py
│   ├── config.py              # Moved from root
│   ├── orchestrator.py        # Agent orchestration engine
│   ├── schema.py              # JSON schema validation
│   └── context.py             # Context window management
│
├── tools/                     # Physics tools (reorganized)
│   ├── __init__.py
│   ├── simulation/            # Event generation tools
│   │   ├── feynrules/
│   │   ├── mg5/
│   │   ├── pythia/
│   │   └── sherpa/
│   ├── analysis/              # Analysis and reconstruction
│   │   ├── kinematics.py
│   │   ├── reconstruction.py
│   │   └── conversions.py
│   ├── literature/            # Literature and data tools
│   │   ├── inspire/
│   │   └── pdg/
│   └── utilities/             # Units, conversions
│       └── units/
│
├── mcp/                       # MCP server (promoted to top-level)
│   ├── __init__.py
│   ├── server_http.py
│   ├── server_stdio.py
│   └── tools.py
│
├── memory/                    # NEW: Agent memory layer
│   ├── __init__.py
│   ├── short_term.py          # In-context conversation memory
│   ├── long_term.py           # Vector store for past analyses
│   └── retrieval.py           # RAG-based memory retrieval
│
├── evaluation/                # NEW: Agent evaluation framework
│   ├── __init__.py
│   ├── metrics.py             # Faithfulness, relevance, accuracy
│   ├── benchmarks/            # Standard HEP analysis benchmarks
│   └── reporter.py            # Evaluation reports
│
├── agents/                    # NEW: Agent definitions
│   ├── __init__.py
│   ├── base_agent.py          # Base agent class
│   ├── router_agent.py        # Query routing (small/fast model)
│   └── specialist_agent.py    # Domain specialist (larger model)
│
├── llm/                       # LLM utilities (kept, cleaned)
│   ├── __init__.py
│   └── utils.py
│
├── prompts/                   # System prompts
│   ├── __init__.py
│   └── hep_bsm/
│
├── tests/                     # Centralized test directory
│   ├── tools/
│   ├── core/
│   ├── memory/
│   └── evaluation/
│
├── examples/                  # Clean examples only
│   ├── workflows/
│   └── notebooks/
│
├── environment.yml
├── requirements.txt
└── README.md
```

---

## 3. Proposed New Tools

### Tool 1 - RAG-Based Literature Retrieval
**Location:** `tools/literature/rag_retrieval.py`

Current INSPIRE search is keyword-based. Replacing with 
semantic retrieval using dense embeddings (Sentence-BERT + FAISS) 
would enable natural language queries over HEP literature.
```python
class RAGLiteratureTool:
    """
    Semantic retrieval over HEP literature corpus.
    Extends current INSPIRE tool with vector search.
    """
    def search(self, query: str, top_k: int = 5) -> list[Paper]:
        # Encode query with Sentence-BERT
        # Search FAISS index of paper embeddings
        # Return ranked relevant papers
        pass
```

**Rationale:** Built and validated a RAG pipeline achieving 
91% factual accuracy on a domain-specific corpus (Kubernetes 
documentation). Same approach applies to HEP literature.

### Tool 2 - Agent Memory with Vector Store
**Location:** `memory/long_term.py`

Enables agents to reference previous analyses, avoiding 
redundant computation and building institutional knowledge.
```python
class LongTermMemory:
    """
    Persistent vector store for past agent analyses.
    Enables retrieval of relevant past work.
    """
    def store(self, analysis_id: str, result: dict) -> None:
        pass
    
    def retrieve(self, query: str, top_k: int = 3) -> list[dict]:
        pass
```

### Tool 3 - Agent Quality Evaluation
**Location:** `evaluation/metrics.py`

RAGAS-inspired evaluation measuring:
- **Tool call accuracy** — did the agent invoke the right tools?
- **Answer faithfulness** — is the answer grounded in tool outputs?
- **Context relevance** — did retrieved literature match the query?
```python
class HEPAgentEvaluator:
    def evaluate(self, 
                 query: str,
                 tool_calls: list,
                 response: str) -> EvaluationResult:
        pass
```

### Tool 4 - Multi-Agent Coordinator
**Location:** `agents/router_agent.py`

Implements hierarchical agent architecture:
- Small fast model handles simple queries directly
- Routes complex queries to specialist agent
- Reduces computational cost for routine analyses
```python
class RouterAgent:
    """
    Lightweight routing agent that classifies query complexity
    and delegates to appropriate specialist agent via MCP.
    """
    def route(self, query: str) -> AgentResponse:
        complexity = self._classify(query)
        if complexity == "simple":
            return self.small_model.generate(query)
        return self.specialist_agent.generate(query)
```

---

## 4. Rationale Summary

| Change | Problem Solved |
|--------|---------------|
| Promote MCP to top-level | MCP is core infrastructure, not an example |
| Extract shared/ from examples/ | Utilities shouldn't depend on examples |
| Centralize tests/ | Consistent testing across all modules |
| Add memory/ layer | Agents can reference past analyses |
| Add evaluation/ layer | Measurable quality improvement over time |
| Add agents/ layer | Multi-agent coordination for complex analyses |
| Reorganize tools/ by domain | Clearer separation of simulation vs analysis |

---

## 5. Implementation Priority

For the GSoC coding period (175 hours):

**Phase 1 (Weeks 1-4):** Restructure — migrate to proposed 
folder structure, update imports, ensure all tests pass.

**Phase 2 (Weeks 5-8):** Memory layer — implement short-term 
and long-term memory with vector store retrieval.

**Phase 3 (Weeks 9-12):** Evaluation framework — RAGAS-style 
metrics for HEP agent quality measurement.

**Phase 4 (Weeks 13-14):** Multi-agent coordinator — router 
agent delegating to specialist via MCP.

---

*Submitted for GSoC 2026 — ML4SCI Organization*
*Contact: ml4-sci@cern.ch*
