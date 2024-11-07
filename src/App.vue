<script setup lang="ts">
import {
    Card,
    CardHeader,
    CardTitle,
    CardDescription,
} from "@/components/ui/card";
import { Label } from "@/components/ui/label";
import {
    Select,
    SelectContent,
    SelectItem,
    SelectTrigger,
    SelectValue,
} from "@/components/ui/select";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import * as vNG from "v-network-graph";
import { ref, reactive, onMounted } from "vue";
import data from "@/assets/data.json";
import { Switch } from "@/components/ui/switch";

// State
const nodeName = ref("");
const networkGraph = ref<vNG.VNetworkGraphInstance>();
const devMode = ref(false);
const algorithm = ref("dijkstra");
const dfsDetour = ref("parking");
const directions = ref<string[]>([]);
const selectedNodes = ref<string[]>(["node1"]);
const selectedEdges = ref<string[]>([]);
const paths = ref<vNG.Paths>({});
const nextNodeIndex = ref(Object.keys(data.nodes).length + 1);
const nextEdgeIndex = ref(Object.keys(data.edges).length + 1);
const currentPathIndex = ref(0);
const allPaths = ref<string[][]>([]);

// Constants
const layers = {
    map: "base",
};

// Graph data
const layouts: vNG.Layouts = reactive(data.layouts);
const nodes: vNG.Nodes = reactive(data.nodes);
const edges: vNG.Edges = reactive(data.edges);

// Graph configuration
const config = reactive(
    vNG.defineConfigs({
        node: {
            selectable: 2,
            draggable: devMode.value,
            normal: {
                color: (node) => {
                    switch (node.category) {
                        case "hall":
                            return "#fb923c";
                        case "parking":
                            return "#60a5fa";
                        case "poi":
                            return "#4ade80";
                        default:
                            return "#fb923c";
                    }
                },
            },
            label: {
                color: "#64748b",
                fontSize: 16,
            },
        },
        edge: {
            normal: {
                color: "#94a3b8",
            },
            selected: {
                color: "#475569",
            },
            label: {
                color: "#475569",
            },
        },
        path: {
            visible: true,
            curveInNode: true,
            path: {
                color: "#ff0000",
                width: 4,
                dasharray: "5",
                animate: true,
            },
        },
        view: {
            autoPanAndZoomOnLoad: false,
        },
    }),
);

// Graph class for pathfinding algorithms
type GraphMap = { [key: string]: { [key: string]: number } };
type EdgeMap = { [key: string]: { [key: string]: string } };

class Graph {
    map: GraphMap;
    edgeMap: EdgeMap;

    private _sorter = (a: string, b: string) => parseFloat(a) - parseFloat(b);

    constructor(edges: vNG.Edges) {
        const map: GraphMap = {};
        const edgeMap: EdgeMap = {};

        Object.entries(edges).forEach(([edgeId, edge]) => {
            const { source, target, cost = 1 } = edge;

            if (!map[source]) map[source] = {};
            if (!map[target]) map[target] = {};
            if (!edgeMap[source]) edgeMap[source] = {};
            if (!edgeMap[target]) edgeMap[target] = {};

            if (map[source][target]) {
                if (map[source][target] > cost) {
                    map[source][target] = cost;
                    map[target][source] = cost;
                    edgeMap[source][target] = edgeId;
                    edgeMap[target][source] = edgeId;
                }
            } else {
                map[source][target] = cost;
                map[target][source] = cost;
                edgeMap[source][target] = edgeId;
                edgeMap[target][source] = edgeId;
            }
        });

        this.map = map;
        this.edgeMap = edgeMap;
    }

    findShortestPath(viaNodes: string[]) {
        return this._findShortestPath(this.map, viaNodes);
    }

    convertNodesToEdges(nodes: string[]): string[] {
        if (nodes.length === 0) return [];

        const edges: string[] = [];
        let prev = nodes[0];

        for (let i = 1; i < nodes.length; i++) {
            const next = nodes[i];
            edges.push(this.edgeMap[prev][next]);
            prev = next;
        }

        return edges;
    }

    private _extractKeys(obj: object) {
        return Object.keys(obj).filter((key) =>
            Object.prototype.hasOwnProperty.call(obj, key),
        );
    }

    private _findPaths(map: GraphMap, start: string, end: string) {
        const costs: { [key: string]: number } = { [start]: 0 };
        const open: { [key: string]: string[] } = { 0: [start] };
        const predecessors: { [key: string]: string } = {};

        const addToOpen = (cost: number, vertex: string) => {
            const key = cost.toString();
            if (!open[key]) open[key] = [];
            open[key].push(vertex);
        };

        while (true) {
            const keys = this._extractKeys(open);
            if (!keys.length) break;

            keys.sort(this._sorter);
            const key = keys[0];
            const bucket = open[key];
            const node = bucket.shift()!;
            const currentCost = parseFloat(key);

            if (!bucket.length) delete open[key];

            for (const [vertex, cost] of Object.entries(map[node] || {})) {
                const totalCost = cost + currentCost;
                const vertexCost = costs[vertex];

                if (vertexCost === undefined || vertexCost > totalCost) {
                    costs[vertex] = totalCost;
                    addToOpen(totalCost, vertex);
                    predecessors[vertex] = node;
                }
            }
        }

        return costs[end] === undefined ? null : predecessors;
    }

    private _extractShortest(
        predecessors: { [key: string]: string },
        end: string,
    ) {
        const nodes = [];
        let current = end;

        while (current !== undefined) {
            nodes.push(current);
            current = predecessors[current];
        }

        return nodes.reverse();
    }

    private _findShortestPath(map: GraphMap, nodes: string[]) {
        nodes = [...nodes];
        const path: string[] = [];
        let start = nodes.shift()!;

        while (nodes.length) {
            const end = nodes.shift()!;
            const predecessors = this._findPaths(map, start, end);

            if (!predecessors) return null;

            const shortest = this._extractShortest(predecessors, end);
            path.push(...(nodes.length ? shortest.slice(0, -1) : shortest));
            start = end;
        }

        return path;
    }

    findPathBFS(viaNodes: string[]): string[] | null {
        if (viaNodes.length < 2) return null;

        const completePath: string[] = [];
        for (let i = 0; i < viaNodes.length - 1; i++) {
            const pathSegment = this._findPathBFS(viaNodes[i], viaNodes[i + 1]);
            if (!pathSegment) return null;

            if (i === 0) {
                completePath.push(...pathSegment);
            } else {
                completePath.push(...pathSegment.slice(1));
            }
        }

        return completePath;
    }

    private _findPathBFS(start: string, end: string): string[] | null {
        const queue = [start];
        const visited = new Set([start]);
        const predecessors: { [key: string]: string } = {};

        while (queue.length) {
            const current = queue.shift()!;

            if (current === end) {
                const path = [];
                let node = end;
                while (node !== start) {
                    path.unshift(node);
                    node = predecessors[node];
                }
                path.unshift(start);
                return path;
            }

            for (const neighbor in this.map[current]) {
                if (!visited.has(neighbor)) {
                    visited.add(neighbor);
                    predecessors[neighbor] = current;
                    queue.push(neighbor);
                }
            }
        }

        return null;
    }

    private _findPathDFS(
        start: string,
        end: string,
        detourType: string | null = null,
        maxPaths: number = 10,
    ): string[][] {
        const visited = new Set<string>();
        const allPaths: string[][] = [];

        const dfs = (current: string, currentPath: string[]) => {
            if (allPaths.length >= maxPaths) return;

            if (current === end) {
                allPaths.push([...currentPath]);
                return;
            }

            const neighbors = Object.keys(this.map[current]);
            if (detourType) {
                neighbors.sort((a, b) => {
                    const aPreferred = nodes[a].category === detourType;
                    const bPreferred = nodes[b].category === detourType;
                    if (aPreferred && !bPreferred) return -1;
                    if (!aPreferred && bPreferred) return 1;
                    return 0;
                });
            }

            for (const neighbor of neighbors) {
                if (!visited.has(neighbor)) {
                    visited.add(neighbor);
                    currentPath.push(neighbor);
                    dfs(neighbor, currentPath);
                    currentPath.pop();
                    visited.delete(neighbor);
                }
            }
        };

        visited.add(start);
        dfs(start, [start]);
        return allPaths;
    }

    findPathDFS(
        viaNodes: string[],
        detourType: string | null = null,
    ): string[][] {
        if (viaNodes.length < 2) return [];

        const completePaths: string[][] = [];
        const firstPaths = this._findPathDFS(
            viaNodes[0],
            viaNodes[1],
            detourType,
        );

        firstPaths.forEach((firstPath) => {
            let isValidPath = true;
            const completePath = [...firstPath];

            for (let i = 2; i < viaNodes.length && isValidPath; i++) {
                const nextPaths = this._findPathDFS(
                    viaNodes[i - 1],
                    viaNodes[i],
                    detourType,
                    1,
                );
                if (nextPaths.length === 0) {
                    isValidPath = false;
                } else {
                    completePath.push(...nextPaths[0].slice(1));
                }
            }

            if (isValidPath) {
                completePaths.push(completePath);
            }
        });

        return completePaths;
    }
}

// Node operations
function addNode() {
    const nodeId = `node${nextNodeIndex.value}`;
    layouts.nodes[nodeId] = { x: 400, y: 400 };
    nodes[nodeId] = { name: nodeName.value };
    nextNodeIndex.value++;
}

function removeNode() {
    selectedNodes.value.forEach((nodeId) => {
        delete nodes[nodeId];
    });
}

// Edge operations
function addEdge() {
    if (selectedNodes.value.length !== 2) return;

    const [source, target] = selectedNodes.value;
    const cost =
        Math.hypot(
            layouts.nodes[source].x - layouts.nodes[target].x,
            layouts.nodes[source].y - layouts.nodes[target].y,
        ) * 2.9;

    const edgeId = `edge${nextEdgeIndex.value}`;
    edges[edgeId] = { source, target, cost };
    nextEdgeIndex.value++;
}

function removeEdge() {
    selectedEdges.value.forEach((edgeId) => {
        delete edges[edgeId];
    });
}

// Graph data export
function getGraphData() {
    const graphData = {
        nodes: { ...nodes },
        edges: { ...edges },
        layouts: { ...layouts },
    };

    const blob = new Blob([JSON.stringify(graphData, null, 2)], {
        type: "application/json",
    });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = "graph-data.json";
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
}

// View initialization
onMounted(() => {
    networkGraph.value?.setViewBox({
        top: -40,
        bottom: 1250,
        left: -250,
        right: 1450,
    });
});

// Mode toggles
function toggleDevMode() {
    devMode.value = !devMode.value;
    config.node!.draggable = devMode.value;
}

// Route finding
function findRoute() {
    const graph = new Graph(data.edges);
    let routeOfNodes: string[] | null = null;
    allPaths.value = [];
    currentPathIndex.value = 0;

    switch (algorithm.value) {
        case "dijkstra":
            routeOfNodes = graph.findShortestPath(selectedNodes.value);
            if (routeOfNodes) allPaths.value = [routeOfNodes];
            break;
        case "bfs":
            routeOfNodes = graph.findPathBFS(selectedNodes.value);
            if (routeOfNodes) allPaths.value = [routeOfNodes];
            break;
        case "dfs":
            allPaths.value = graph.findPathDFS(
                selectedNodes.value,
                dfsDetour.value === "none" ? null : dfsDetour.value,
            );
            // Sort paths by length (shortest first)
            allPaths.value.sort((a, b) => a.length - b.length);
            routeOfNodes = allPaths.value[0] || null;
            break;
    }

    if (routeOfNodes) {
        updatePathDisplay();
    }
}

function getDirections() {
    paths.value.shortestPath.edges.forEach((edgeId, index) => {
        const edge = edges[edgeId];
        if (!edge) return;

        const prevEdgeId = paths.value.shortestPath.edges[index - 1];
        const prevEdge = prevEdgeId ? edges[prevEdgeId] : null;

        if (prevEdge && edge.source !== prevEdge.target) {
            [edge.source, edge.target] = [edge.target, edge.source];
        }

        directions.value.push(
            `Walk from ${nodes[edge.source].name} to ${nodes[edge.target].name}`,
        );
    });
}

function updatePathDisplay() {
    if (allPaths.value.length === 0) return;

    const currentPath = allPaths.value[currentPathIndex.value];
    const routeOfEdges = new Graph(edges).convertNodesToEdges(currentPath);
    paths.value = { shortestPath: { edges: routeOfEdges } };
    directions.value = [];
    getDirections();
}

function nextPath() {
    if (currentPathIndex.value < allPaths.value.length - 1) {
        currentPathIndex.value++;
        updatePathDisplay();
    }
}

function previousPath() {
    if (currentPathIndex.value > 0) {
        currentPathIndex.value--;
        updatePathDisplay();
    }
}
</script>

<template>
    <div class="min-h-screen bg-slate-50 p-8">
        <!-- Main Container -->
        <div class="max-w-7xl mx-auto space-y-6">
            <!-- Header Card -->
            <Card class="w-full">
                <CardHeader>
                    <CardTitle>School Navigation System</CardTitle>
                    <CardDescription
                        >Find the best route between locations</CardDescription
                    >
                    <div class="flex items-center gap-2 mt-2">
                        <Label class="font-medium">Dev Mode:</Label>
                        <Switch
                            :checked="devMode"
                            @update:checked="toggleDevMode"
                        />
                    </div>
                </CardHeader>
            </Card>

            <!-- Map Container Card -->
            <div
                class="flex items-center gap-6 p-4 bg-white rounded-lg shadow-sm"
                v-if="devMode"
            >
                <div class="flex items-center gap-2">
                    <Label class="font-medium">Node:</Label>
                    <Button variant="outline" size="sm" @click="addNode"
                        >Add Node</Button
                    >
                    <Button
                        variant="outline"
                        size="sm"
                        :disabled="selectedNodes.length == 0"
                        @click="removeNode"
                        >Remove Node</Button
                    >
                </div>
                <div class="flex items-center gap-2">
                    <Label class="font-medium">Edge:</Label>
                    <Button
                        variant="outline"
                        size="sm"
                        :disabled="selectedNodes.length != 2"
                        @click="addEdge"
                        >Add Edge</Button
                    >
                    <Button
                        variant="outline"
                        size="sm"
                        :disabled="selectedEdges.length == 0"
                        @click="removeEdge"
                        >Remove Edge</Button
                    >
                </div>
                <div>
                    <Button variant="default" size="sm" @click="getGraphData"
                        >Save Graph</Button
                    >
                </div>
                <div>
                    <Label class="font-medium">Node Name:</Label>
                    <Input v-model="nodeName" />
                </div>
            </div>

            <Card class="w-full h-[1000px]">
                <VNetworkGraph
                    ref="networkGraph"
                    v-model:selected-nodes="selectedNodes"
                    v-model:selected-edges="selectedEdges"
                    v-model:layouts="layouts"
                    :nodes="nodes"
                    :edges="edges"
                    :paths="paths"
                    :layers="layers"
                    :configs="config"
                >
                    <template #map>
                        <image
                            href="./assets/img/csuf-parking-map.svg"
                            alt="Map"
                            x="0"
                            y="0"
                            width="1000px"
                        />
                    </template>
                    <template #edge-label="{ edge, ...slotProps }">
                        <v-edge-label
                            :text="Math.round(edge.cost) + ' ft'"
                            align="center"
                            vertical-align="below"
                            v-bind="slotProps"
                        />
                    </template>
                </VNetworkGraph>
            </Card>
            <Card
                v-if="algorithm === 'dfs' && allPaths.length > 1"
                class="w-full p-6"
            >
                <div class="flex items-center justify-between">
                    <div class="text-sm text-slate-600">
                        Path {{ currentPathIndex + 1 }} of {{ allPaths.length }}
                    </div>
                    <div class="flex gap-2">
                        <Button
                            variant="outline"
                            size="sm"
                            @click="previousPath"
                            :disabled="currentPathIndex === 0"
                        >
                            Previous Path
                        </Button>
                        <Button
                            variant="outline"
                            size="sm"
                            @click="nextPath"
                            :disabled="currentPathIndex === allPaths.length - 1"
                        >
                            Next Path
                        </Button>
                    </div>
                </div>
            </Card>
            <!-- Navigation Controls Card -->
            <Card class="w-full p-6">
                <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
                    <!-- Algorithm Selection -->
                    <div class="space-y-2">
                        <Label class="text-md font-medium" for="algorithm"
                            >Algorithm</Label
                        >
                        <Select v-model="algorithm">
                            <SelectTrigger>
                                <SelectValue placeholder="Select algorithm" />
                            </SelectTrigger>
                            <SelectContent>
                                <SelectItem value="dijkstra"
                                    >Dijkstra's Algorithm</SelectItem
                                >
                                <SelectItem value="bfs"
                                    >Breadth First Search</SelectItem
                                >
                                <SelectItem value="dfs"
                                    >Depth First Search</SelectItem
                                >
                            </SelectContent>
                        </Select>

                        <!-- DFS Options -->
                        <div v-if="algorithm === 'dfs'" class="mt-4">
                            <Label class="text-md font-medium" for="dfsDetour"
                                >Detour Preference</Label
                            >
                            <Select v-model="dfsDetour">
                                <SelectTrigger>
                                    <SelectValue
                                        placeholder="Select detour preference"
                                    />
                                </SelectTrigger>
                                <SelectContent>
                                    <SelectItem value="parking"
                                        >Visit Parking Lots</SelectItem
                                    >
                                    <SelectItem value="poi"
                                        >Visit Points of Interest</SelectItem
                                    >
                                    <SelectItem value="none"
                                        >Direct Path</SelectItem
                                    >
                                </SelectContent>
                            </Select>
                        </div>
                    </div>

                    <!-- Start Location -->
                    <div class="space-y-2">
                        <Label for="start">Start Location</Label>
                        <Select v-model="selectedNodes[0]">
                            <SelectTrigger>
                                <SelectValue
                                    placeholder="Select start location"
                                />
                            </SelectTrigger>
                            <SelectContent>
                                <SelectItem
                                    v-for="(node, id) in nodes"
                                    :value="id"
                                    :key="id"
                                    >{{ node.name }}</SelectItem
                                >
                            </SelectContent>
                        </Select>
                    </div>

                    <!-- End Location -->
                    <div class="space-y-2">
                        <Label for="end">End Location</Label>
                        <Select v-model="selectedNodes[1]">
                            <SelectTrigger>
                                <SelectValue placeholder="Select destination" />
                            </SelectTrigger>
                            <SelectContent>
                                <SelectItem
                                    v-for="(node, id) in nodes"
                                    :value="id"
                                    :key="id"
                                    >{{ node.name }}</SelectItem
                                >
                            </SelectContent>
                        </Select>
                    </div>
                </div>

                <!-- Find Route Button -->
                <div class="mt-6 flex justify-end">
                    <Button variant="default" @click="findRoute"
                        >Find Route</Button
                    >
                </div>
            </Card>

            <Card class="w-full p-6">
                <div class="flex items-center gap-2">
                    <div class="flex-1">
                        <h3 class="font-medium">Directions</h3>
                        <div
                            v-if="paths.shortestPath"
                            class="mt-2 text-sm text-slate-600"
                        >
                            <p>
                                Total Distance:
                                {{
                                    Math.round(
                                        Object.values(
                                            paths.shortestPath.edges,
                                        ).reduce(
                                            (acc, edgeId) =>
                                                acc +
                                                (edges[edgeId]?.cost || 0),
                                            0,
                                        ),
                                    )
                                }}
                                feet
                            </p>
                            <div class="mt-2">
                                <p
                                    v-for="(direction, index) in directions"
                                    :key="direction"
                                >
                                    {{ index + 1 }}. {{ direction }}
                                </p>
                            </div>
                        </div>
                        <div v-else class="mt-2 text-sm text-slate-600">
                            Select start and end locations to see directions
                        </div>
                    </div>
                </div>
            </Card>
        </div>
    </div>
</template>
