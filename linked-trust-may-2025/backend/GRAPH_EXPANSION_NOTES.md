# Graph Expansion Design Notes

## Original Design (from main branch)

### Key Insights

1. **`getClaimGraph(claimId)`** endpoint (`/api/claim_graph/:claimId`)
   - Finds ALL nodes connected to a specific claim
   - Returns ALL edges for those nodes, not just claim-specific edges
   - This gives full context for each node and enables exploration

2. **No arbitrary depth limits**
   - Gets all edges of discovered nodes in one query
   - Frontend controls the exploration depth through user interaction

3. **Expansion Mechanism**
   - User clicks a node → Frontend identifies new claims to explore
   - Call API with new claim IDs
   - Frontend merges new nodes/edges into existing graph

### Original