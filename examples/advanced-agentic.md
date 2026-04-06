# Advanced Example: Agentic Workflows with CO-STA-RG

ตัวอย่างระดับสูงของการใช้ CO-STA-RG Framework สำหรับ Agentic Workflows 
(Prompt ที่ AI จะส่งต่อไปยัง Agent อื่น)

---

## 🎯 Use Case: Security Audit Workflow

สถานการณ์: คุณต้องการ AI ที่:
1. **รับ** Python code จาก Git repository
2. **วิเคราะห์** เพื่อหา security vulnerabilities
3. **ส่ง output เป็น JSON** ให้ Tool ขั้นต่อไป (SIEM system)
4. **Auto-trigger** remediation workflow

---

## 📋 Agentic Prompt

```xml
<Prompt>
  <Title>Security Code Analyzer for CI/CD Pipeline</Title>
  <Description>Automated security vulnerability scanner for Python code in CI/CD</Description>
  <Confidence>0.88</Confidence>
  <Tags>security,code-analysis,agentic,ci-cd,automation</Tags>
  <Creator>security-team</Creator>
  <TargetAgent>SIEM-Remediation-Agent</TargetAgent>
</Prompt>

---

## Prompt

### Context (บริบท)
You are a Security Scanning Agent embedded in a CI/CD pipeline 
for a financial services company (SOC 2 certified).
You receive Python code from pull requests and must identify 
security vulnerabilities before deployment.
Your analysis feeds directly into an automated remediation engine.
You work 24/7 with zero human review - your classifications trigger automatic actions.
Mistakes are costly.

### Objective (วัตถุประสงค์)
For the provided Python code:
1. Scan for OWASP Top 10 vulnerabilities
2. Classify each finding by:
   - Severity: CRITICAL | HIGH | MEDIUM | LOW
   - Category: injection, auth, crypto, etc
   - CVSS Score: 0-10
3. Output exactly N findings per severity level (or 0 if none)
4. Provide remediation code snippet for each
5. Include CWE identifier and reference

Format compliance: MUST be valid JSON for downstream parsing

### Style (สไตล์)
Persona: Automated Security Scanner
- Be precise and unambiguous
- Think like compliance checker, not friendly assistant
- Every field must be machine-parseable
- No filler text

### Tone (น้ำเสียง)
Tone: Rigorous and unforgiving
- Err on side of caution (flag anything remotely suspicious)
- No benefit-of-doubt
- Assume attacker will exploit every weakness

### Audience (กลุ่มเป้าหมาย)
Audience: Automated systems + Security team
- PRIMARY: SIEM remediation engine (requires JSON)
- SECONDARY: Human security reviewer (reads JSON)

### Response (รูปแบบผลลัพธ์)
Format: JSON (MUST be valid and parseable by downstream tools)

REQUIRED structure:

```json
{
  "scan_id": "uuid",
  "timestamp": "ISO-8601",
  "file_scanned": "filepath",
  "total_findings": {
    "CRITICAL": 0,
    "HIGH": 0,
    "MEDIUM": 0,
    "LOW": 0
  },
  "findings": [
    {
      "id": "finding-uuid",
      "severity": "CRITICAL|HIGH|MEDIUM|LOW",
      "category": "injection|auth|crypto|xxe|broken-access|xss|insecure-deser|auth-failure|soft-logging",
      "title": "string",
      "description": "string (2-3 sentences max)",
      "line_number": integer,
      "vulnerable_code": "exact code snippet",
      "remediation": "exact fixed code snippet",
      "cwe_id": "CWE-XXXX",
      "cvss_score": 0.0-10.0,
      "cvss_vector": "string",
      "owasp_mapping": "string",
      "false_positive_likelihood": "very-low|low|medium|high",
      "automated_fix_available": boolean,
      "requires_human_review": boolean,
      "reference": "url"
    }
  ],
  "scan_status": "PASS|FAIL|REVIEW_REQUIRED",
  "recommendation": "string (remediation action)"
}
```

### Grammar & Grounding (ไวยากรณ์และการอ้างอิง)

**Syntax Rules:**
- Valid JSON ONLY - no markdown, no comments
- All string fields must use escaped quotes
- Timestamps in ISO 8601 format
- No trailing commas

**Security Grounding:**
- Only identify vulnerabilities actually present (testable)
- Reference OWASP Top 10 2021 official documentation
- CWE IDs must be accurate and real
- CVSS scores must follow CVSS 3.1 specification
- If unsure: mark false_positive_likelihood as "high", not invented

**Fail-Safe:**
- If JSON malformed: output with error flag
- If code unparseable: report "parse_error" field
- If ambiguous finding: include in output but flag requires_human_review=true

---

## 📊 Example: Real Vulnerable Code

### Input Code (Python):

```python
# User authentication endpoint
@app.route('/login', methods=['POST'])
def login():
    username = request.args.get('username')
    password = request.args.get('password')
    
    # Vulnerability 1: SQL Injection
    query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
    user = db.execute(query)
    
    # Vulnerability 2: Hardcoded secret
    SECRET_KEY = "my-super-secret-key"
    token = jwt.encode({'user': username}, SECRET_KEY)
    
    return jsonify({'token': token})
```

### Expected Output (JSON):

```json
{
  "scan_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2026-04-06T10:30:00Z",
  "file_scanned": "src/auth.py",
  "total_findings": {
    "CRITICAL": 2,
    "HIGH": 0,
    "MEDIUM": 0,
    "LOW": 0
  },
  "findings": [
    {
      "id": "finding-sql-001",
      "severity": "CRITICAL",
      "category": "injection",
      "title": "SQL Injection via User Input",
      "description": "Username and password parameters are directly concatenated into SQL query without escaping. Attacker can inject SQL commands.",
      "line_number": 7,
      "vulnerable_code": "query = f\"SELECT * FROM users WHERE username='{username}' AND password='{password}'\"",
      "remediation": "query = \"SELECT * FROM users WHERE username=? AND password=?\"\nuser = db.execute(query, (username, password))",
      "cwe_id": "CWE-89",
      "cvss_score": 9.8,
      "cvss_vector": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H",
      "owasp_mapping": "A1:2021 – Injection",
      "false_positive_likelihood": "very-low",
      "automated_fix_available": true,
      "requires_human_review": false,
      "reference": "https://owasp.org/www-community/attacks/SQL_Injection"
    },
    {
      "id": "finding-secrets-001",
      "severity": "CRITICAL",
      "category": "crypto",
      "title": "Hardcoded Secret in Source Code",
      "description": "JWT secret key is hardcoded in source code. Exposed in version control. Must use environment variables.",
      "line_number": 11,
      "vulnerable_code": "SECRET_KEY = \"my-super-secret-key\"",
      "remediation": "SECRET_KEY = os.environ.get('JWT_SECRET_KEY')\nif not SECRET_KEY:\n    raise ValueError('JWT_SECRET_KEY not set')",
      "cwe_id": "CWE-798",
      "cvss_score": 9.1,
      "cvss_vector": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N",
      "owasp_mapping": "A2:2021 – Cryptographic Failures",
      "false_positive_likelihood": "very-low",
      "automated_fix_available": true,
      "requires_human_review": false,
      "reference": "https://owasp.org/www-project-top-ten/2021/A02_2021-Cryptographic_Failures"
    }
  ],
  "scan_status": "FAIL",
  "recommendation": "Block deployment. Fix CRITICAL vulnerabilities before merge. Consider automated remediation."
}
```

---

## ⚙️ How This Integrates with Other Agents

### Workflow Chain:

```
1. Developer submits PR
   ↓
2. CO-STA-RG Prompt (This one)
   ├─ Scans code
   └─ Outputs JSON
   ↓
3. SIEM Remediation Agent receives JSON
   ├─ Parses findings
   ├─ AUTO: Apply fixes (if automated_fix_available=true)
   ├─ NOTIFY: Security team (if requires_human_review=true)
   └─ BLOCK: Deployment (if severity=CRITICAL)
   ↓
4. Security Team Reviews
   ├─ Approves fixes
   └─ Merges PR
```

---

## 🔑 Key Points for Agentic Prompts

### ✅ DO:

1. **Strict JSON Format**
   - Downstream agents expect exact format
   - Invalid JSON breaks automation
   
2. **Machine-Parseable Field Names**
   - Use consistent naming
   - No variations or alternatives

3. **Numeric IDs & References**
   - Always include UUID or ticket ID
   - Traceability for auditing

4. **Fail-Safe Fallbacks**
   - What happens if input is invalid?
   - Graceful degradation vs hard failures

5. **Classification Fields**
   - Use enums (CRITICAL|HIGH|etc)
   - Never free-text classifications

### ❌ DON'T:

1. ❌ Output commentary or "by the way..."
2. ❌ Mix formats (JSON + markdown)
3. ❌ Include unstructured data
4. ❌ Assume downstream agent is intelligent
5. ❌ Skip error handling

---

## 📈 Testing Agentic Prompts

### Step 1: Validate JSON Output
```bash
curl -X POST your-ai-endpoint \n  -H "Content-Type: application/json" \n  -d '{"prompt": "..."}' | jq '.' # Check valid JSON
```
### Step 2: Test with Downstream Agent
```bash
# Feed output to SIEM system
python siem_ingestion.py output.json
```
### Step 3: Edge Cases
- What if code is empty?
- What if 0 vulnerabilities found?
- What if multiple findings same line?

### Step 4: Monitor in Production
- Track parsing errors
- Monitor false positives
- Measure automation success rate

---

## 🎓 Advanced Concepts

### Confidence Scores in Agentic Context

```json
{
  "false_positive_likelihood": "very-low",  // Agentic confidence
  "automated_fix_available": true,           // Can downstream agent fix?
  "requires_human_review": false             // Escalation flag
}
```

### Traceability

Each finding has:
- `scan_id` - Which scan produced this?
- `finding-uuid` - Unique ID for tracking
- `reference` - Link to documentation
- `line_number` - Exact location in code

### Error Handling

```json
{
  "scan_status": "FAIL",
  "error": {
    "code": "PARSE_ERROR",
    "message": "Could not parse Python syntax",
    "fallback_action": "ESCALATE_TO_HUMAN"
  }
}
```

---

## 💡 Real-World Lessons

### What We Learned:

1. **Specificity is critical** - "security findings" became structured JSON
2. **Integration matters** - Output format must match downstream systems
3. **Fail-safes save lives** - Always have escalation path
4. **Traceability = Compliance** - Every decision must be auditable
5. **Automation ≠ Perfection** - Keep false_positive flags

---

## 🔗 Related Resources

- [PRINCIPLES.md](../core-frameworks/PRINCIPLES.md) - Core principles
- [templates/prompt-template.md](../templates/prompt-template.md) - Build your own
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CVSS Calculator](https://www.first.org/cvss/calculator/3.1)

---

_Agentic workflows demand precision. CO-STA-RG delivers it._