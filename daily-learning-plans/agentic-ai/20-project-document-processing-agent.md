# Agentic AI — Day 20: Project — Document Processing Pipeline Agent
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Build a Multi-Agent Pipeline That Ingests, Classifies, Extracts, and Processes Documents**

Enterprises process thousands of documents daily: invoices, contracts, support tickets, compliance forms. This agent pipeline automatically classifies documents, extracts structured data, validates it, and routes it to the right system. It's a real-world use case that combines classification, structured extraction, RAG, and multi-agent orchestration.

---

## 2. Architecture

```
Document arrives (PDF, email, form)
       │
       ▼
┌──────────────┐
│ Classifier   │  → invoice / contract / support_ticket / compliance
│ Agent        │
└──────┬───────┘
       │
  ┌────┴────────────┬──────────────┐
  ▼                 ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Invoice  │  │ Contract │  │ Ticket   │
│ Extractor│  │ Extractor│  │ Extractor│
└─────┬────┘  └─────┬────┘  └─────┬────┘
      │              │              │
      ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│Validator │  │Validator │  │Validator │
└─────┬────┘  └─────┬────┘  └─────┬────┘
      │              │              │
      └──────────────┼──────────────┘
                     ▼
              ┌──────────────┐
              │ Router       │ → ERP / CRM / Ticketing system
              └──────────────┘
```

---

## 3. Implementation (Python — LangGraph)

### 3.1 Document State

```python
from typing import TypedDict, Annotated, Literal, Optional
from pydantic import BaseModel

class DocumentState(TypedDict):
    raw_text: str
    filename: str
    doc_type: str               # invoice, contract, support_ticket
    extracted_data: dict
    validation_errors: list[str]
    is_valid: bool
    routing_destination: str    # erp, crm, jira
    messages: Annotated[list, add_messages]
```

### 3.2 Classifier Agent

```python
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-sonnet-4-20250514")

class DocumentClassification(BaseModel):
    doc_type: Literal["invoice", "contract", "support_ticket", "compliance", "unknown"]
    confidence: float
    reasoning: str

def classify_document(state: DocumentState) -> dict:
    result = model.with_structured_output(DocumentClassification).invoke([
        ("system", """Classify this document into one of:
        - invoice: bills, payment requests, purchase orders
        - contract: agreements, NDAs, SLAs, terms of service
        - support_ticket: customer complaints, bug reports, feature requests
        - compliance: audit reports, regulatory filings, policy documents
        - unknown: cannot determine"""),
        ("user", f"Filename: {state['filename']}\n\nContent:\n{state['raw_text'][:3000]}")
    ])
    return {"doc_type": result.doc_type}
```

### 3.3 Specialised Extractors

```python
class InvoiceData(BaseModel):
    vendor_name: str
    invoice_number: str
    invoice_date: str
    due_date: str
    line_items: list[dict]       # [{description, quantity, unit_price, total}]
    subtotal: float
    tax: float
    total: float
    currency: str
    payment_terms: str

class ContractData(BaseModel):
    parties: list[str]
    contract_type: str           # NDA, SLA, employment, vendor
    effective_date: str
    expiry_date: str
    key_terms: list[str]
    obligations: list[str]
    termination_clause: str
    governing_law: str

class TicketData(BaseModel):
    customer_name: str
    customer_email: str
    issue_type: str              # bug, feature_request, complaint, question
    priority: str                # low, medium, high, urgent
    summary: str
    steps_to_reproduce: str
    expected_behavior: str
    actual_behavior: str

def extract_invoice(state: DocumentState) -> dict:
    data = model.with_structured_output(InvoiceData).invoke([
        ("system", "Extract all invoice fields from this document. Be precise with numbers."),
        ("user", state["raw_text"])
    ])
    return {"extracted_data": data.model_dump()}

def extract_contract(state: DocumentState) -> dict:
    data = model.with_structured_output(ContractData).invoke([
        ("system", "Extract all contract details. Pay attention to dates, obligations, and termination clauses."),
        ("user", state["raw_text"])
    ])
    return {"extracted_data": data.model_dump()}

def extract_ticket(state: DocumentState) -> dict:
    data = model.with_structured_output(TicketData).invoke([
        ("system", "Extract support ticket details. Classify priority based on urgency language."),
        ("user", state["raw_text"])
    ])
    return {"extracted_data": data.model_dump()}
```

### 3.4 Validators

```python
def validate_invoice(state: DocumentState) -> dict:
    data = state["extracted_data"]
    errors = []
    # Business rules
    if not data.get("invoice_number"):
        errors.append("Missing invoice number")
    if data.get("total", 0) <= 0:
        errors.append("Invalid total amount")
    if data.get("total") and data.get("subtotal") and data.get("tax"):
        expected = round(data["subtotal"] + data["tax"], 2)
        if abs(data["total"] - expected) > 0.01:
            errors.append(f"Total mismatch: {data['total']} != {data['subtotal']} + {data['tax']}")
    # Line items check
    if data.get("line_items"):
        calc_total = sum(item.get("total", 0) for item in data["line_items"])
        if abs(calc_total - data.get("subtotal", 0)) > 0.01:
            errors.append(f"Line items don't sum to subtotal: {calc_total} != {data['subtotal']}")

    return {"validation_errors": errors, "is_valid": len(errors) == 0}

def validate_contract(state: DocumentState) -> dict:
    data = state["extracted_data"]
    errors = []
    if len(data.get("parties", [])) < 2:
        errors.append("Contract must have at least 2 parties")
    if not data.get("effective_date"):
        errors.append("Missing effective date")
    if not data.get("termination_clause"):
        errors.append("Missing termination clause — legal review required")
    return {"validation_errors": errors, "is_valid": len(errors) == 0}

def validate_ticket(state: DocumentState) -> dict:
    data = state["extracted_data"]
    errors = []
    if not data.get("customer_email"):
        errors.append("Missing customer email — cannot respond")
    if not data.get("summary"):
        errors.append("Missing issue summary")
    return {"validation_errors": errors, "is_valid": len(errors) == 0}
```

### 3.5 Router

```python
def route_document(state: DocumentState) -> dict:
    routing = {
        "invoice": "erp",       # → SAP, NetSuite, QuickBooks
        "contract": "crm",      # → Salesforce, HubSpot
        "support_ticket": "jira" # → Jira, Zendesk
    }
    destination = routing.get(state["doc_type"], "manual_review")
    if not state["is_valid"]:
        destination = "manual_review"
    return {"routing_destination": destination}
```

### 3.6 Build the Graph

```python
from langgraph.graph import StateGraph, START, END

def route_by_type(state: DocumentState) -> str:
    return state["doc_type"]

graph = StateGraph(DocumentState)

graph.add_node("classify", classify_document)
graph.add_node("extract_invoice", extract_invoice)
graph.add_node("extract_contract", extract_contract)
graph.add_node("extract_ticket", extract_ticket)
graph.add_node("validate_invoice", validate_invoice)
graph.add_node("validate_contract", validate_contract)
graph.add_node("validate_ticket", validate_ticket)
graph.add_node("route", route_document)

graph.add_edge(START, "classify")
graph.add_conditional_edges("classify", route_by_type, {
    "invoice": "extract_invoice",
    "contract": "extract_contract",
    "support_ticket": "extract_ticket",
    "unknown": "route"   # Send to manual review
})

graph.add_edge("extract_invoice", "validate_invoice")
graph.add_edge("extract_contract", "validate_contract")
graph.add_edge("extract_ticket", "validate_ticket")

graph.add_edge("validate_invoice", "route")
graph.add_edge("validate_contract", "route")
graph.add_edge("validate_ticket", "route")
graph.add_edge("route", END)

doc_pipeline = graph.compile()
```

---

## 4. Test with Sample Documents

```python
invoice_text = """
INVOICE #INV-2026-0042
Date: 2026-04-01
Due: 2026-05-01

From: Acme Cloud Services
To: TechCorp Inc.

Items:
- Cloud Hosting (March 2026): 3 units × $499.00 = $1,497.00
- SSL Certificates: 5 units × $29.99 = $149.95
- Support Plan: 1 unit × $199.00 = $199.00

Subtotal: $1,845.95
Tax (10%): $184.60
Total: $2,030.55
Payment Terms: Net 30
"""

result = doc_pipeline.invoke({
    "raw_text": invoice_text, "filename": "invoice_march.pdf",
    "doc_type": "", "extracted_data": {}, "validation_errors": [],
    "is_valid": False, "routing_destination": "", "messages": []
})

print(f"Type: {result['doc_type']}")
print(f"Extracted: {json.dumps(result['extracted_data'], indent=2)}")
print(f"Valid: {result['is_valid']}, Errors: {result['validation_errors']}")
print(f"Route to: {result['routing_destination']}")
```

---

## 5. Hands-On Exercise

1. Build the full document processing pipeline.
2. Create 3 sample documents (invoice, contract, support ticket) as text.
3. Run each through the pipeline and verify: classification → extraction → validation → routing.
4. Introduce an error (wrong total) and verify the validator catches it.
5. Add a "compliance" document type with its own extractor and validator.

---

## 6. Key Takeaways

1. **Structured extraction (Pydantic models) makes LLM output machine-readable** — directly usable by downstream systems.
2. **Validation is separate from extraction** — the LLM extracts, business rules validate.
3. **Classification routes to specialists** — each document type has its own extractor and rules.
4. **Invalid documents go to manual review** — never auto-process data you can't validate.

---

*Day 20. Document processing is where agents meet the enterprise — structured, validated, and routed.*
