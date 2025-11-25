import sqlglot
import re
from typing import Dict, List, Tuple

def identify_partition_keys(sql: str) -> List[Dict]:
    """
    Identify likely partition keys from SQL query.
    Returns list of columns with scores.
    """
    tree = sqlglot.parse_one(sql)
    
    # Extract WHERE clause (focus on main query, not subqueries for now)
    where_clause = tree.find(sqlglot.expressions.Where)
    
    if not where_clause:
        return []  # No WHERE clause = can't determine partition keys
    
    # Track columns and their characteristics
    column_scores = {}
    condition_order = 0
    
    # Walk through all conditions in WHERE
    for condition in where_clause.find_all(sqlglot.expressions.Condition):
        condition_order += 1
        
        # Analyze different condition types
        if isinstance(condition, sqlglot.expressions.In):
            # Pattern: col IN (val1, val2, ...)
            column = condition.this
            if isinstance(column, sqlglot.expressions.Column):
                col_name = column.sql()
                values = condition.expressions  # The list of values
                
                score = 0
                score += 5  # IN clause bonus
                score += analyze_column_name(col_name)
                score += analyze_values(values)
                if condition_order == 1:
                    score += 1  # First condition bonus
                
                if col_name not in column_scores or score > column_scores[col_name]['score']:
                    column_scores[col_name] = {
                        'score': score,
                        'filter_type': 'IN',
                        'num_values': len(values)
                    }
        
        elif isinstance(condition, sqlglot.expressions.EQ):
            # Pattern: col = value
            left = condition.this
            right = condition.expression
            
            if isinstance(left, sqlglot.expressions.Column):
                col_name = left.sql()
                value = right
                
                score = 0
                score += 3  # Equality bonus
                score += analyze_column_name(col_name)
                score += analyze_single_value(value)
                if condition_order == 1:
                    score += 1
                
                if col_name not in column_scores or score > column_scores[col_name]['score']:
                    column_scores[col_name] = {
                        'score': score,
                        'filter_type': 'EQUALS',
                        'value': value.sql()
                    }
        
        elif isinstance(condition, sqlglot.expressions.Between):
            # Pattern: col BETWEEN val1 AND val2
            column = condition.this
            if isinstance(column, sqlglot.expressions.Column):
                col_name = column.sql()
                
                score = 0
                score += 2  # BETWEEN bonus
                score += analyze_column_name(col_name)
                if condition_order == 1:
                    score += 1
                
                if col_name not in column_scores or score > column_scores[col_name]['score']:
                    column_scores[col_name] = {
                        'score': score,
                        'filter_type': 'BETWEEN'
                    }
    
    # Convert to sorted list
    results = []
    for col_name, info in column_scores.items():
        results.append({
            'column': col_name,
            'score': info['score'],
            'filter_type': info.get('filter_type'),
            'likely_partition_key': info['score'] >= 8,
            'possibly_partition_key': 5 <= info['score'] < 8
        })
    
    # Sort by score descending
    results.sort(key=lambda x: x['score'], reverse=True)
    
    return results


def analyze_column_name(col_name: str) -> int:
    """
    Score based on column name patterns.
    """
    col_upper = col_name.upper()
    score = 0
    
    # Date/time indicators (strong signal)
    if any(keyword in col_upper for keyword in ['DATE', 'DT', 'TIME', 'YEAR', 'MONTH', 'DAY']):
        score += 3
    
    # Categorical partitioning indicators (weak signal)
    elif any(keyword in col_upper for keyword in ['REGION', 'COUNTRY', 'TYPE', 'CATEGORY', 'STATUS']):
        score += 1
    
    return score


def analyze_values(values: List) -> int:
    """
    Score based on value patterns (for IN clause).
    """
    if not values or len(values) < 3:
        return 0  # Too few values
    
    score = 0
    
    # Check first few values for patterns
    sample_values = [v.sql() for v in values[:5]]
    
    # Check for 8-digit integers (YYYYMMDD format)
    if all(re.match(r'^\d{8}$', str(v).strip("'\"")) for v in sample_values):
        score += 2
    
    # Check for date strings (YYYY-MM-DD)
    elif all(re.match(r'^\d{4}-\d{2}-\d{2}', str(v).strip("'\"")) for v in sample_values):
        score += 2
    
    # Check for short categorical values (regions, types, etc.)
    elif all(len(str(v).strip("'\"")) <= 10 for v in sample_values):
        score += 1
    
    return score


def analyze_single_value(value) -> int:
    """
    Score based on single value pattern (for equality).
    """
    if not value:
        return 0
    
    val_str = value.sql().strip("'\"")
    score = 0
    
    # 8-digit integer (YYYYMMDD)
    if re.match(r'^\d{8}$', val_str):
        score += 2
    
    # Date string
    elif re.match(r'^\d{4}-\d{2}-\d{2}', val_str):
        score += 2
    
    # Short categorical
    elif len(val_str) <= 10:
        score += 1
    
    return score


# Example usage:
if __name__ == "__main__":
    sql1 = """
    SELECT order_id, amount
    FROM orders
    WHERE TRADE_DATE IN (20250313, 20251106, 20210309, 20251027)
      AND customer_id = 'ABC123'
      AND amount > 1000
    """
    
    results = identify_partition_keys(sql1)
    
    print("Partition Key Analysis:")
    print("="*70)
    for result in results:
        print(f"\nColumn: {result['column']}")
        print(f"  Score: {result['score']}")
        print(f"  Filter Type: {result['filter_type']}")
        if result['likely_partition_key']:
            print(f"  ✅ Very likely partition key")
        elif result['possibly_partition_key']:
            print(f"  ⚠️  Possibly partition key")
        else:
            print(f"  ❌ Unlikely partition key")
    
    print("\n" + "="*70)
    
    # Test with your real query
    sql2 = """
    WITH CACHE_OM_CR_SECURITY_FACT AS (
        SELECT 'Credit_risk' SRC_SYSTEM, 
               security_id Flow_entity_id,
               cr_security_sk Flow_entity_sk
        FROM gfolyrsk_cr_managed.OM_CR_SECURITY_FACT 
        WHERE dwh_business_date = 20250119
    ) 
    SELECT src_system, flow_entity_id, flow_entity_sk 
    FROM CACHE_OM_CR_SECURITY_FACT 
    WHERE SECURITY_ID_TYPE = 'SMCP'
    """
    
    print("\n\nAnalyzing second query...")
    results2 = identify_partition_keys(sql2)
    
    print("Partition Key Analysis:")
    print("="*70)
    for result in results2:
        print(f"\nColumn: {result['column']}")
        print(f"  Score: {result['score']}")
        print(f"  Filter Type: {result['filter_type']}")
        if result['likely_partition_key']:
            print(f"  ✅ Very likely partition key")
        elif result['possibly_partition_key']:
            print(f"  ⚠️  Possibly partition key")
        else:
            print(f"  ❌ Unlikely partition key")
