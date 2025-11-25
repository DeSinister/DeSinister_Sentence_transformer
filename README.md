import sqlglot
import re

def identify_partition_keys(sql: str):
    """Recursively find partition keys at any nesting level."""
    tree = sqlglot.parse_one(sql)
    column_scores = {}
    
    # Recursively analyze the entire tree
    analyze_node_recursive(tree, column_scores)
    
    # Convert to results
    results = []
    for col_name, info in column_scores.items():
        results.append({
            'column': col_name,
            'score': info['score'],
            'filter_type': info['filter_type'],
            'value': info['value'],
            'likely_partition_key': info['score'] >= 8
        })
    
    results.sort(key=lambda x: x['score'], reverse=True)
    return results


def analyze_node_recursive(node, column_scores, depth=0, is_cte=False):
    """
    Recursively walk through SQL tree and analyze WHERE clauses at any level.
    """
    # Prevent infinite recursion
    if depth > 20:
        return
    
    # Check if this node is a CTE
    if isinstance(node, sqlglot.expressions.CTE):
        is_cte = True
    
    # Check if this node has a WHERE clause
    if isinstance(node, (sqlglot.expressions.Select, sqlglot.expressions.Subquery)):
        where_clause = node.find(sqlglot.expressions.Where)
        if where_clause:
            analyze_where_clause(where_clause, column_scores, is_cte)
    
    # Recursively process children
    for child in node.iter_expressions():
        analyze_node_recursive(child, column_scores, depth + 1, is_cte)


def analyze_where_clause(where_clause, column_scores, is_cte):
    """Analyze WHERE clause for partition key patterns."""
    for node in where_clause.walk():
        if isinstance(node, sqlglot.expressions.In):
            # col IN (val1, val2, ...)
            column = node.this
            if isinstance(column, sqlglot.expressions.Column):
                col_name = column.sql()
                values = node.expressions
                
                score = 5  # IN bonus
                score += score_column_name(col_name)
                score += score_values(values)
                if is_cte:
                    score += 1
                
                update_score(column_scores, col_name, score, 'IN', f"{len(values)} values")
        
        elif isinstance(node, sqlglot.expressions.EQ):
            # col = value
            left = node.this
            right = node.expression
            
            if isinstance(left, sqlglot.expressions.Column):
                col_name = left.sql()
                value = right.sql() if right else ''
                
                score = 3  # EQ bonus
                score += score_column_name(col_name)
                score += score_single_value(value)
                if is_cte:
                    score += 1
                
                update_score(column_scores, col_name, score, 'EQUALS', value)
        
        elif isinstance(node, sqlglot.expressions.Between):
            # col BETWEEN x AND y
            column = node.this
            if isinstance(column, sqlglot.expressions.Column):
                col_name = column.sql()
                
                score = 2  # BETWEEN bonus
                score += score_column_name(col_name)
                if is_cte:
                    score += 1
                
                update_score(column_scores, col_name, score, 'BETWEEN', '')


def update_score(column_scores, col_name, score, filter_type, value):
    """Update score if higher than existing."""
    if col_name not in column_scores or score > column_scores[col_name]['score']:
        column_scores[col_name] = {
            'score': score,
            'filter_type': filter_type,
            'value': value
        }


def score_column_name(col_name: str) -> int:
    """Score based on column name."""
    col_upper = col_name.upper()
    if any(k in col_upper for k in ['DATE', 'DT', 'TIME', 'YEAR', 'MONTH', 'DAY']):
        return 3
    elif any(k in col_upper for k in ['REGION', 'COUNTRY', 'TYPE', 'CATEGORY']):
        return 1
    return 0


def score_values(values) -> int:
    """Score based on multiple values (IN clause)."""
    if not values or len(values) < 3:
        return 0
    
    sample_values = [v.sql() for v in values[:5]]
    
    # 8-digit dates
    if all(re.match(r'^\d{8}$', str(v).strip("'\"")) for v in sample_values):
        return 2
    # Date strings
    elif all(re.match(r'^\d{4}-\d{2}-\d{2}', str(v).strip("'\"")) for v in sample_values):
        return 2
    # Categorical
    elif all(len(str(v).strip("'\"")) <= 10 for v in sample_values):
        return 1
    
    return 0


def score_single_value(value: str) -> int:
    """Score based on single value."""
    val_str = value.strip("'\"")
    
    if re.match(r'^\d{8}$', val_str):
        return 2
    elif re.match(r'^\d{4}-\d{2}-\d{2}', val_str):
        return 2
    elif len(val_str) <= 10:
        return 1
    
    return 0


# Test
sql = """
with CACHE_OM_DEBT_COMMON_BALANCE as ( 
    select * from (
        select * from table1 where DWH_BUSINESS_DATE = 20251121
    ) where OTHER_COL = 'value'
)
select * from CACHE_OM_DEBT_COMMON_BALANCE
"""

results = identify_partition_keys(sql)

print("Partition Key Analysis:")
print("="*70)
for result in results:
    print(f"\nColumn: {result['column']}")
    print(f"  Score: {result['score']}")
    print(f"  Filter: {result['filter_type']}")
    print(f"  Value: {result['value']}")
    if result['likely_partition_key']:
        print(f"  ✅ LIKELY PARTITION KEY")
