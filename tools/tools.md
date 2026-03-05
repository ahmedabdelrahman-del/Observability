# AWS Cost Explorer Tool Handler Flow

This document explains the internal flow of the function:

makeCostByServiceHandler()

This handler is responsible for processing tool calls from the AI agent, validating the input, querying AWS Cost Explorer, and returning structured cost data.

---

# 1. Handler Creation

Function:

makeCostByServiceHandler(ce any)

This function is a **handler factory**.  
It receives the AWS Cost Explorer client and returns a tool handler function.

Purpose:
- Attach AWS client dependency to the handler
- Allow dependency injection (useful for testing with mock clients)

Flow:

Tool Registration
        ↓
ToolCostByService()
        ↓
makeCostByServiceHandler(ce)
        ↓
Handler function returned

---

# 2. Handler Execution

The handler function signature:

(ctx context.Context, raw json.RawMessage) (any, *protocol.RPCError)

Inputs:
- `ctx` → request context (timeouts, cancellation)
- `raw` → raw JSON arguments coming from the AI tool call

Outputs:
- `any` → response data
- `*protocol.RPCError` → structured error if something fails

---

# 3. Parse Incoming Arguments

First step: convert JSON arguments into Go struct.

Example code:

json.Unmarshal(raw, &args)

Target struct:

type CostByServiceArgs struct {
  Start       string
  End         string
  Service     string
  Granularity string
  Metric      string
}

Example tool call from AI:

{
 "start": "2026-03-01",
 "end": "2026-03-05",
 "service": "AmazonEC2"
}

If parsing fails, return:

protocol.InvalidParams("invalid arguments")

---

# 4. Validate Dates

Dates must follow:

YYYY-MM-DD

Example validation:

time.Parse("2006-01-02", args.Start)

Checks performed:

- Start date must be valid
- End date must be valid
- End must be after Start

Invalid example:

start: 2026-03-10  
end:   2026-03-05

This returns:

InvalidParams("end must be after start")

---

# 5. Apply Default Values

If optional parameters are missing:

Granularity default → MONTHLY  
Metric default → UnblendedCost

Example:

User request:

{
 "start": "2026-03-01",
 "end": "2026-03-05"
}

Resolved values:

granularity = MONTHLY  
metric = UnblendedCost

---

# 6. Validate AWS Client

The handler receives the AWS client as `any`.

We convert it into the required interface:

ceClient, ok := ce.(costExplorerCostAndUsageAPI)

Why?

To ensure the object supports:

GetCostAndUsage()

If not valid:

InternalError("cost explorer client not configured")

---

# 7. Validate Granularity

Allowed values:

DAILY  
MONTHLY

Invalid example:

granularity = WEEKLY

Result:

InvalidParams("granularity must be DAILY or MONTHLY")

---

# 8. Build AWS Cost Explorer Request

Create request object:

costexplorer.GetCostAndUsageInput

Example:

TimePeriod
Granularity
Metrics

Example request:

{
  TimePeriod: { Start: "2026-03-01", End: "2026-03-05" },
  Granularity: DAILY,
  Metrics: ["UnblendedCost"]
}

---

# 9. Service Filter Logic

Two possible behaviors:

## Case A: Service specified

Example:

service = AmazonEC2

Request adds filter:

Filter:
  Dimension: SERVICE
  Values: ["AmazonEC2"]

AWS returns cost only for that service.

---

## Case B: Service not specified

Request groups costs by service:

GroupBy:
  Dimension: SERVICE

AWS returns cost per service.

Example response concept:

EC2 → $120  
S3 → $30  
CloudWatch → $10

---

# 10. Call AWS API

Execute request:

ceClient.GetCostAndUsage(ctx, input)

Possible failure reasons:

- AWS credentials issue
- API throttling
- incorrect query parameters

Errors are wrapped using:

protocol.WrapInternalError()

---

# 11. Parse AWS Response

AWS response contains:

ResultsByTime

Each element represents one time window.

Example structure:

ResultsByTime
 ├── TimePeriod
 ├── Total
 └── Groups

Two parsing paths exist:

### If service filter used

Read cost from:

row.Total[metric]

---

### If grouped by service

Loop through:

row.Groups

Extract:

service name
cost amount

Store results in:

byService map

Example:

byService:

AmazonEC2 → 120  
AmazonS3 → 30

---

# 12. Build Period Breakdown

For each time period:

Create:

{
 "start": "2026-03-01",
 "end": "2026-03-02",
 "total": 25.10
}

All periods stored in:

periods array

---

# 13. Build Final Response

Example final response:

{
 "start": "2026-03-01",
 "end": "2026-03-05",
 "granularity": "DAILY",
 "metric": "UnblendedCost",
 "currency": "USD",
 "total": 150.75,
 "periods": [...],
 "byService": [...]
}

---

# 14. Service Cost Sorting

Services are sorted by highest cost first.

Example:

1. AmazonEC2 → $120  
2. AmazonS3 → $30  
3. CloudWatch → $10

This makes it easier for AI to detect **top cost drivers**.

---

# 15. Numeric Cleanup

Function:

roundTo(value, decimals)

Used to avoid floating precision issues.

Example:

12.345678912 → 12.345679

---

# 16. Safe Pointer Reading

AWS SDK often returns pointer values:

*string

Helper function:

valueOrEmpty()

Prevents nil pointer errors.

Example:

nil → ""

---

# 17. Complete Execution Flow

User question
        ↓
AI selects tool
        ↓
Tool receives JSON arguments
        ↓
Parse arguments
        ↓
Validate dates and inputs
        ↓
Apply defaults
        ↓
Build AWS request
        ↓
Call Cost Explorer
        ↓
Parse AWS response
        ↓
Aggregate totals
        ↓
Sort services
        ↓
Return structured JSON result