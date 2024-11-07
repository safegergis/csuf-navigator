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
const nodeName = ref("");
const networkGraph = ref<vNG.VNetworkGraphInstance>();
const devMode = ref(false);
const layers = {
    map: "base",
};
const algorithm = ref("dijkstra");

const config = reactive(
    vNG.defineConfigs({
        node: {
            selectable: 2,
            draggable: devMode.value,
            normal: {
                color: (node) => {
                    switch (node.category) {
                        case "hall":
                            return "#fb923c"; // Orange for buildings/halls
                        case "parking":
                            return "#60a5fa"; // Blue for parking lots
                        case "poi":
                            return "#4ade80"; // Green for points of interest
                        default:
                            return "#fb923c";
                    }
                },
            },
            label: {
                color: "#64748b", // Light slate for better contrast on any background
                fontSize: 16,
            },
        },
        edge: {
            normal: {
                color: "#94a3b8", // Slate gray for edges
            },
            selected: {
                color: "#475569", // Darker slate when selected
            },
            label: {
                color: "#475569", // Slate for edge labels
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
const layouts: vNG.Layouts = reactive(data.layouts);
const nodes: vNG.Nodes = reactive(data.nodes);
const edges: vNG.Edges = reactive(data.edges);
const nextNodeIndex = ref(Object.keys(nodes).length + 1);
const nextEdgeIndex = ref(Object.keys(edges).length + 1);

const selectedNodes = ref<string[]>(["node1"]);
const selectedEdges = ref<string[]>([]);
type GraphMap = { [key: string]: { [key: string]: number } };
type EdgeMap = { [key: string]: { [key: string]: string } };
const paths = ref<vNG.Paths>({});

class Graph {
    map: GraphMap;
    edgeMap: EdgeMap;

    _sorter = function (a: string, b: string) {
        return parseFloat(a) - parseFloat(b);
    };

    constructor(edges: vNG.Edges) {
        // var map = {a:{b:3,c:1},b:{a:2,c:1},c:{a:4,b:1}},
        const map: GraphMap = {};
        const edgeMap: EdgeMap = {};
        Object.entries(edges).forEach(([edgeId, edge]) => {
            const source = edge.source;
            const target = edge.target;
            const cost = edge?.cost ?? 1;
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
        const edges: string[] = [];
        if (nodes.length === 0) {
            return [];
        }
        let prev = nodes[0];
        for (let i = 1; i < nodes.length; i++) {
            const next = nodes[i];
            edges.push(this.edgeMap[prev][next]);
            prev = next;
        }
        return edges;
    }

    _extractKeys(obj: object) {
        const keys = [];
        for (const key in obj) {
            if (Object.prototype.hasOwnProperty.call(obj, key)) {
                keys.push(key);
            }
        }
        return keys;
    }

    _findPaths(map: GraphMap, start: string, end: string) {
        const costs: { [key: string]: number } = {};
        const open: { [key: string]: string[] } = { 0: [start] };
        const predecessors: { [key: string]: string } = {};
        let keys;

        const addToOpen = function (cost: number, vertex: string) {
            const key = "" + cost;
            if (!open[key]) {
                open[key] = [];
            }
            open[key].push(vertex);
        };

        costs[start] = 0;

        while (open) {
            if (!(keys = this._extractKeys(open)).length) {
                break;
            }

            keys.sort(this._sorter);

            const key = keys[0];
            const bucket = open[key];
            const node = bucket.shift() || "";
            const currentCost = parseFloat(key);
            const adjacentNodes = map[node] || {};

            if (!bucket.length) {
                delete open[key];
            }

            for (const vertex in adjacentNodes) {
                if (
                    Object.prototype.hasOwnProperty.call(adjacentNodes, vertex)
                ) {
                    const cost = adjacentNodes[vertex];
                    const totalCost = cost + currentCost;
                    const vertexCost = costs[vertex];

                    if (vertexCost === undefined || vertexCost > totalCost) {
                        costs[vertex] = totalCost;
                        addToOpen(totalCost, vertex);
                        predecessors[vertex] = node;
                    }
                }
            }
        }

        if (costs[end] === undefined) {
            return null;
        } else {
            return predecessors;
        }
    }

    _extractShortest(predecessors: { [key: string]: string }, end: string) {
        const nodes = [];
        let u = end;

        while (u !== undefined) {
            nodes.push(u);
            u = predecessors[u];
        }

        nodes.reverse();
        return nodes;
    }

    _findShortestPath(map: GraphMap, nodes: string[]) {
        nodes = [...nodes]; // copy
        let start = nodes.shift() || "";
        let end: string;
        let predecessors;
        const path: string[] = [];
        let shortest;

        while (nodes.length) {
            end = nodes.shift() || "";
            predecessors = this._findPaths(map, start, end);

            if (predecessors) {
                shortest = this._extractShortest(predecessors, end);
                if (nodes.length) {
                    path.push.apply(path, shortest.slice(0, -1));
                } else {
                    return path.concat(shortest);
                }
            } else {
                return null;
            }

            start = end;
        }
    }
}

const dijkstra = () => {
    const graph = new Graph(data.edges);
    const routeOfNodes = graph.findShortestPath([
        selectedNodes.value[0],
        selectedNodes.value[1],
    ]);
    if (routeOfNodes) {
        const routeOfEdges = graph.convertNodesToEdges(routeOfNodes);
        paths.value = { shortestPath: { edges: routeOfEdges } };
    }
    console.log(paths.value);
};

function addNode() {
    const nodeId = `node${nextNodeIndex.value}`;
    const name = nodeName.value;
    layouts.nodes[nodeId] = { x: 400, y: 400 };
    nodes[nodeId] = { name };
    nextNodeIndex.value++;
}

function removeNode() {
    for (const nodeId of selectedNodes.value) {
        delete nodes[nodeId];
    }
}

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
    for (const edgeId of selectedEdges.value) {
        delete edges[edgeId];
    }
}

// Get graph data
const getGraphData = () => {
    const graphData = {
        nodes: { ...nodes },
        edges: { ...edges },
        layouts: { ...layouts },
    };
    const jsonData = JSON.stringify(graphData, null, 2);
    // Save to file using Blob and download
    const blob = new Blob([jsonData], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = "graph-data.json";
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    URL.revokeObjectURL(url);
    return graphData;
};
onMounted(() => {
    networkGraph.value?.setViewBox({
        top: -40,
        bottom: 1250,
        left: -250,
        right: 1450,
    });
});
const toggleDevMode = () => {
    devMode.value = !devMode.value;
    config.node!.draggable = devMode.value;
};
const findRoute = () => {
    switch (algorithm.value) {
        case "dijkstra":
            dijkstra();
            break;
    }
};
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

            <!-- Navigation Controls Card -->
            <Card class="w-full p-6">
                <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
                    <!-- Algorithm Selection -->
                    <div class="space-y-2">
                        <Label class="text-md font-medium" for="algorithm">
                            Algorithm
                        </Label>
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
                    <Button variant="default" @click="findRoute">
                        Find Route
                    </Button>
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
                            <div class="mt-2"></div>
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
