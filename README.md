KEYWORD_EXTRACTION_PROMPT = """You are an expert at analyzing database query requests. Your task is to extract all potential keywords and entities from a user's natural language query, regardless of word order or format.

USER QUERY:
{user_query}

TASK:
Extract all potential keywords and classify them into categories. Users may phrase queries in any order, so you must identify entities based on meaning, not position.

IMPORTANT CONTEXT:
- The target column in the prompt is often a BUSINESS TERM (e.g., "FRS Business Unit") that will later map to a TECHNICAL COLUMN NAME (e.g., "FRS BU" or "FRS_BU")
- The table in the prompt is often a BUSINESS TERM (e.g., "Corporate Loans") that will later map to a TECHNICAL TABLE NAME (e.g., "STD_CLN_INB_DL")
- Extract the BUSINESS TERMS as they appear in the prompt - the fuzzy matching to technical names happens in the next step
- Focus on identifying what the user is referring to, even if it's phrased in business language

IMPORTANT CONTEXT:
- The target column name in the prompt may be a BUSINESS TERM (e.g., "FRS Business Unit") that maps to a TECHNICAL COLUMN NAME (e.g., "FRS BU")
- The table name in the prompt may be a BUSINESS TERM (e.g., "Corporate Loans") that maps to a TECHNICAL TABLE NAME (e.g., "STD_CLN_INB_DL")
- Extract the BUSINESS TERMS from the prompt - the mapping to technical names happens in the next step
- Focus on identifying what the user is referring to, even if it's phrased in business language

EXTRACTION RULES:
1. Extract ALL noun phrases and important terms that could refer to database entities
2. Consider business terminology, abbreviations, and technical names
3. Don't rely on word order - "column from table" and "from table column" mean the same thing
4. Extract terms that might be:
   - Column names (e.g., "FRS Business Unit", "customer_id", "amount")
   - Table names (e.g., "Corporate Loans", "customer_table", "orders")
   - Schema names (e.g., "Olympus Composition Layer", "sales_schema")
   - Domain names (e.g., "Customer Management", "Sales Domain")
   - Query actions (e.g., "produce error", "show", "find", "validate")
   - Conditions (e.g., "is null", "is blank", "equals", "greater than")

OUTPUT FORMAT (JSON):
{{
    "target_column": {{
        "text": "the column that the rule/validation applies to",
        "confidence": 0.0-1.0,
        "context_clues": ["why this is the target column"]
    }},
    "dataset_source": {{
        "schema": "schema name if specified (e.g., 'Olympus Risk managed Schema')",
        "table": "table name if specified",
        "domain": "domain name if specified",
        "confidence": 0.0-1.0
    }},
    "condition_columns": [
         {{
            "text": "column used in WHERE clause or filtering condition",
            "condition": "the condition (e.g., 'is null', 'equals X')",
            "confidence": 0.0-1.0
        }}
    ],
    "keywords": [
        {{
            "text": "exact phrase from query",
            "type": "table|schema|domain|action|condition_phrase|ambiguous",
            "confidence": 0.0-1.0,
            "context_clues": ["reasoning for classification"]
        }}
    ],
    "query_intent": "brief description of what user wants to do",
    "extracted_phrases": [
        "all noun phrases and important terms"
    ]
}}

CLASSIFICATION GUIDELINES:

**CRITICAL DISTINCTION - TARGET COLUMN vs CONDITION COLUMNS:**
- **TARGET COLUMN**: The PRIMARY column that the rule/validation needs to be applied ON. This is the column being validated/checked.
  - Examples: "FRS Business Unit" in "Produce Error if FRS Business Unit is null"
  - Indicators: Column mentioned with validation action ("is null", "is blank", "produce error for [X]")
  - This is the MOST IMPORTANT column to identify - it's what the rule is about!
  
- **CONDITION COLUMNS**: Columns used in WHERE clauses for filtering/conditions (NOT the target of validation)
  - Examples: "where status = 'active'" → "status" is a condition column, NOT the target
  - Indicators: Appears after "where", "and", "or" in filtering contexts
  - These are used to FILTER the dataset, not to validate

**DATASET SOURCE:**
- **Schema indicators**: "use dataset from [X] schema", "from [X] schema", "in [X] Layer", terms with "Schema" or "Layer"
- **Table indicators**: "from [table]", "in [table]", "use [table]"
- **Domain indicators**: High-level business domains, organizational units

**OTHER ENTITIES:**
- **Action indicators**: Verbs like "produce", "show", "find", "validate", "check", "generate"
- **Condition phrases**: "is null", "is blank", "equals", "not equal", comparison operators (these describe HOW to validate, not which column)
- **Ambiguous**: When unsure, mark as ambiguous - we'll search in multiple entity types

**PRIORITY:**
1. FIRST identify the TARGET COLUMN (the one the rule applies to)
2. THEN identify dataset source (schema/table/domain)
3. THEN identify condition columns (if any, used in WHERE clauses)
4. Extract other keywords for context

EXAMPLES:

Query: "Produce Error if FRS Business Unit for Corporate Loans is null or Blank in Olympus Composition Layer"
Output:
{{
    "target_column": {{
        "text": "FRS Business Unit",
        "confidence": 0.95,
        "context_clues": ["This is the column being validated - appears with 'is null or Blank' validation", "The rule applies TO this column"]
    }},
    "dataset_source": {{
        "schema": "Olympus Composition Layer",
        "table": "Corporate Loans",
        "domain": null,
        "confidence": 0.9
    }},
    "condition_columns": [],
    "keywords": [
        {{
            "text": "Corporate Loans",
            "type": "table",
            "confidence": 0.8,
            "context_clues": ["business entity name", "likely a table containing loan data"]
        }},
        {{
            "text": "Olympus Composition Layer",
            "type": "schema",
            "confidence": 0.95,
            "context_clues": ["contains 'Layer' which often indicates schema", "contextual location 'in [X]'"]
        }},
        {{
            "text": "null or Blank",
            "type": "condition_phrase",
            "confidence": 1.0,
            "context_clues": ["explicit validation condition - describes HOW to validate target column"]
        }},
        {{
            "text": "Produce Error",
            "type": "action",
            "confidence": 1.0,
            "context_clues": ["explicit action verb"]
        }}
    ],
    "query_intent": "Validate that FRS Business Unit column (TARGET - business term) is not null or blank in Corporate Loans table (business term) within Olympus Composition Layer schema, and produce an error if validation fails",
    "extracted_phrases": ["FRS Business Unit", "Corporate Loans", "Olympus Composition Layer", "null", "Blank", "Error"]
}}

NOTE: "FRS Business Unit" is a BUSINESS TERM extracted from prompt. It will be matched to technical column name (e.g., "FRS BU" or "FRS_BU") in the next step.
NOTE: "Corporate Loans" is a BUSINESS TERM extracted from prompt. It will be matched to technical table name (e.g., "STD_CLN_INB_DL") in the next step.

Query: "from sales schema produce error for customer_id where amount is null"
Output:
{{
    "target_column": {{
        "text": "customer_id",
        "confidence": 0.95,
        "context_clues": ["Appears after 'for' which indicates the target of validation", "The rule applies TO customer_id"]
    }},
    "dataset_source": {{
        "schema": "sales",
        "table": null,
        "domain": null,
        "confidence": 0.9
    }},
    "condition_columns": [
        {{
            "text": "amount",
            "condition": "is null",
            "confidence": 0.9,
            "context_clues": ["Appears in WHERE clause context", "Used to FILTER, not the target of validation"]
        }}
    ],
    "keywords": [
        {{
            "text": "sales",
            "type": "schema",
            "confidence": 0.9,
            "context_clues": ["appears after 'from' and before 'schema'"]
        }},
        {{
            "text": "is null",
            "type": "condition_phrase",
            "confidence": 1.0,
            "context_clues": ["Condition for filtering, not validation target"]
        }},
        {{
            "text": "produce error",
            "type": "action",
            "confidence": 1.0,
            "context_clues": ["explicit action"]
        }}
    ],
    "query_intent": "Produce error for customer_id (TARGET) in sales schema WHERE amount (condition column) is null",
    "extracted_phrases": ["sales", "customer_id", "amount", "null", "error"]
}}

Query: "use the dataset from Olympus Risk managed Schema and produce error if FRS Business Unit is null"
Output:
{{
    "target_column": {{
        "text": "FRS Business Unit",
        "confidence": 0.95,
        "context_clues": ["This is the column being validated - appears with 'is null' validation", "The rule applies TO this column"]
    }},
    "dataset_source": {{
        "schema": "Olympus Risk managed Schema",
        "table": null,
        "domain": null,
        "confidence": 0.95,
        "context_clues": ["Explicitly stated 'use dataset from [X] Schema'"]
    }},
    "condition_columns": [],
    "keywords": [
        {{
            "text": "Olympus Risk managed Schema",
            "type": "schema",
            "confidence": 0.95,
            "context_clues": ["Explicit dataset source specification"]
        }},
        {{
            "text": "is null",
            "type": "condition_phrase",
            "confidence": 1.0,
            "context_clues": ["Validation condition for target column"]
        }},
        {{
            "text": "produce error",
            "type": "action",
            "confidence": 1.0,
            "context_clues": ["explicit action"]
        }}
    ],
    "query_intent": "Use dataset from Olympus Risk managed Schema and validate that FRS Business Unit (TARGET) is not null, produce error if validation fails",
    "extracted_phrases": ["Olympus Risk managed Schema", "FRS Business Unit", "null", "error"]
}}

Query: "in Corporate Loans table from Olympus schema, check Customer ID where Status equals Active and produce error if Amount is blank"
Output:
{{
    "target_column": {{
        "text": "Amount",
        "confidence": 0.95,
        "context_clues": ["This is the column being validated - 'is blank' validation applies to Amount", "The rule applies TO Amount"]
    }},
    "dataset_source": {{
        "schema": "Olympus",
        "table": "Corporate Loans",
        "domain": null,
        "confidence": 0.9
    }},
    "condition_columns": [
        {{
            "text": "Customer ID",
            "condition": "where Status equals Active",
            "confidence": 0.8,
            "context_clues": ["Appears in WHERE clause context", "Used for filtering, not validation target"]
        }},
        {{
            "text": "Status",
            "condition": "equals Active",
            "confidence": 0.9,
            "context_clues": ["Explicitly in WHERE clause 'where Status equals Active'", "Used to FILTER, not the target"]
        }}
    ],
    "keywords": [
        {{
            "text": "Corporate Loans",
            "type": "table",
            "confidence": 0.9
        }},
        {{
            "text": "Olympus",
            "type": "schema",
            "confidence": 0.9
        }},
        {{
            "text": "is blank",
            "type": "condition_phrase",
            "confidence": 1.0,
            "context_clues": ["Validation condition for target column Amount"]
        }},
        {{
            "text": "produce error",
            "type": "action",
            "confidence": 1.0
        }}
    ],
    "query_intent": "In Corporate Loans table from Olympus schema, validate Amount (TARGET) is not blank WHERE Status equals Active (condition columns), produce error if validation fails",
    "extracted_phrases": ["Corporate Loans", "Olympus", "Customer ID", "Status", "Active", "Amount", "blank", "error"]
}}

Query: "FRS Business Unit should not be null for records in Corporate Loans"
Output:
{{
    "target_column": {{
        "text": "FRS Business Unit",
        "confidence": 0.9,
        "context_clues": ["Column being validated - 'should not be null' applies to this", "The rule applies TO FRS Business Unit"]
    }},
    "dataset_source": {{
        "schema": null,
        "table": "Corporate Loans",
        "domain": null,
        "confidence": 0.8
    }},
    "condition_columns": [],
    "keywords": [
        {{
            "text": "Corporate Loans",
            "type": "table",
            "confidence": 0.8
        }},
        {{
            "text": "should not be null",
            "type": "condition_phrase",
            "confidence": 1.0
        }}
    ],
    "query_intent": "Validate that FRS Business Unit (TARGET) is not null in Corporate Loans table",
    "extracted_phrases": ["FRS Business Unit", "Corporate Loans", "null"]
}}

Query: "validate Account Number where Region is North America in Sales Domain"
Output:
{{
    "target_column": {{
        "text": "Account Number",
        "confidence": 0.9,
        "context_clues": ["Column being validated - 'validate' action applies to this", "The rule applies TO Account Number"]
    }},
    "dataset_source": {{
        "schema": null,
        "table": null,
        "domain": "Sales Domain",
        "confidence": 0.85
    }},
    "condition_columns": [
        {{
            "text": "Region",
            "condition": "is North America",
            "confidence": 0.9,
            "context_clues": ["Appears in WHERE clause 'where Region is North America'", "Used for filtering, not validation target"]
        }}
    ],
    "keywords": [
        {{
            "text": "Sales Domain",
            "type": "domain",
            "confidence": 0.85
        }},
        {{
            "text": "validate",
            "type": "action",
            "confidence": 1.0
        }}
    ],
    "query_intent": "Validate Account Number (TARGET) WHERE Region (condition column) is North America in Sales Domain",
    "extracted_phrases": ["Account Number", "Region", "North America", "Sales Domain"]
}}

CRITICAL RULES:
1. **ALWAYS identify the TARGET COLUMN first** - This is the column the rule/validation applies TO
   - NOT columns used in WHERE clauses for filtering
   - The target column is what needs to be validated/checked
   
2. **Distinguish TARGET from CONDITION columns:**
   - TARGET: "produce error for [X]" or "[X] is null" → [X] is the TARGET
   - CONDITION: "where [Y] = 'value'" → [Y] is a condition column (used for filtering)
   
3. **Identify dataset source:**
   - "use dataset from [X] Schema" → schema is dataset source
   - "from [X] schema" → schema is dataset source
   - "in [X] Layer" → schema is dataset source
   
4. **Extract EVERY potentially relevant term** - it's better to over-extract than miss something
5. **When in doubt, mark as "ambiguous"** - we'll search in multiple places
6. **Consider abbreviations and business terminology**
7. **Don't assume format** - users can phrase things in any order
8. **Look for context clues, not just position**

REMEMBER: 
- The TARGET COLUMN is the MOST IMPORTANT - it's what the rule is about!
- Extract BUSINESS TERMS as they appear in the prompt (e.g., "FRS Business Unit", "Corporate Loans")
- These business terms will be matched to technical names (e.g., "FRS BU", "STD_CLN_INB_DL") in the next step using fuzzy matching
- Don't try to convert business terms to technical names here - just extract what the user said

Now extract keywords from the user query above."""
