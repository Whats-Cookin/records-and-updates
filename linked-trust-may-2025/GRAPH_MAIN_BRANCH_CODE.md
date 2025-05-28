# Graph Code from Main Branch

## api.dao.ts - getClaimGraph method

```typescript
getClaimGraph = async (claimId: string | number) => {
  console.log("In getClaimGraph");
  const numericClaimId = typeof claimId === "string" ? parseInt(claimId, 10) : claimId;

  // First find the nodes involved with this claim
  const nodes = await prisma.node.findMany({
    where: {
      OR: [
        {
          edgesFrom: {
            some: {
              claimId: numericClaimId,
            },
          },
        },
        {
          edgesTo: {
            some: {
              claimId: numericClaimId,
            },
          },
        },
      ],
    },
    include: {
      // Include ALL edges connected to these nodes
      edgesFrom: {
        include: {
          claim: true,
          startNode: true,
          endNode: true,
        },
      },
      edgesTo: {
        include: {
          claim: true,
          startNode: true,
          endNode: true,
        },
      },
    },
  });

  // Debug logging
  console.log(`Found ${nodes.length} nodes for claim ${numericClaimId}`);
  nodes.forEach((node) => {
    console.log(`Node ${node.id} (${node.name}):`);
    console.log(`  ${node.edgesFrom.length} outgoing edges`);
    console.log(`  ${node.edgesTo.length} incoming edges`);
  });

  return {
    nodes,
    count: nodes.length,
  };
};
```

## Route setup

```typescript
// From api.route.ts
router.get("/claim_graph/:claimId", claimGraph);

// From api.controller.ts
export const claimGraph = async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { claimId } = req.params;
    const result = await nodeDao.getClaimGraph(claimId);
    res.status(200).json(result);
    return;
  } catch (err) {
    passToExpressErrorHandler(err, next);
  }
};
```

## Key Design Elements

1. **No `/api/graph` endpoint without params** - Everything requires a starting point

2. **Returns ALL edges for discovered nodes** - Not filtered by claim
   - This is crucial for expansion
   - Allows seeing full context of each node
   - Frontend can discover new paths to explore

3. **Simple structure** - Just returns nodes array with all their edges included

4. **Expansion would work by**:
   - Frontend tracks which nodes/claims have been fetched
   - Click node → find unfetched claims from its edges
   - Call `/claim_graph/:claimId` for each new claim
   - Merge results into existing graph

## Other relevant endpoints

### getNodeById
```typescript
getNodeById = async (nodeId: number) => {
  return await prisma.node.findUnique({
    where: {
      id: Number(nodeId),
    },
    include: {
      edgesFrom: {
        select: {
          id: true,
          claimId: true,
          startNodeId: true,
          endNodeId: true,
          label: true,
          thumbnail: true,
          claim: true,
          endNode: true,
          startNode: true,
        },
      },
      edgesTo: {
        select: {
          id: true,
          claimId: true,
          startNodeId: true,
          endNodeId: true,
          label: true,
          thumbnail: true,
          claim: true,
          endNode: true,
          startNode: true,
        },
      },
    },
  });
};
```

### Node search
```typescript
searchNodes = async (search: string, page: number, limit: number) => {
  const query: Prisma.NodeWhereInput = {
    OR: [
      { id: { equals: parseInt(search, 10) } },
      { name: { contains: search, mode: "insensitive" } },
      { descrip: { contains: search, mode: "insensitive" } },
      { nodeUri: { contains: search, mode: "insensitive" } },
    ],
  };

  const nodes = await prisma.node.findMany({
    skip: (Number(page) - 1) * Number(limit),
    take: Number(limit) ? Number(limit) : undefined,
    where: query,
    include: {
      edgesFrom: {
        select: {
          id: true,
          claimId: true,
          startNodeId: true,
          endNodeId: true,
          label: true,
          thumbnail: true,
          claim: true,
          endNode: true,
          startNode: true,
        },
      },
      edgesTo: {
        select: {
          id: true,
          claimId: true,
          startNodeId: true,
          endNodeId: true,
          label: true,
          thumbnail: true,
          claim: true,
          endNode: true,
          startNode: true,
        },
      },
    },
  });

  const count = await prisma.node.count({ where: query });

  return { nodes, count };
};
```

## Notes on Frontend Integration

The frontend would need to:
1. Track fetched claim IDs
2. On node click, extract unfetched claim IDs from edges
3. Batch fetch new claims
4. Merge nodes/edges, deduplicating by ID
5. Update graph visualization with new data

The key is that each node comes with ALL its edges, so you can see what other claims/nodes are connected without having to fetch them first.
