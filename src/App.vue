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
const nodeName = ref("");
const networkGraph = ref<vNG.VNetworkGraphInstance>();
const layers = {
    map: "base",
};
const config = vNG.defineConfigs({
    node: {
        selectable: 2,
    },
    edge: {
        selectable: true,
    },
    view: {
        autoPanAndZoomOnLoad: false,
    },
});
const layouts: vNG.Layouts = reactive({
    nodes: {},
});
const nodes: vNG.Nodes = reactive({});
const edges: vNG.Edges = reactive({});
const nextNodeIndex = ref(Object.keys(nodes).length + 1);
const nextEdgeIndex = ref(Object.keys(edges).length + 1);

const selectedNodes = ref<string[]>([]);
const selectedEdges = ref<string[]>([]);

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
                </CardHeader>
            </Card>

            <!-- Map Container Card -->
            <div
                class="flex items-center gap-6 p-4 bg-white rounded-lg shadow-sm"
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
                        <Select>
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
                        <Select>
                            <SelectTrigger>
                                <SelectValue
                                    placeholder="Select start location"
                                />
                            </SelectTrigger>
                            <SelectContent>
                                <SelectItem value="placeholder"
                                    >Select start location...</SelectItem
                                >
                            </SelectContent>
                        </Select>
                    </div>

                    <!-- End Location -->
                    <div class="space-y-2">
                        <Label for="end">End Location</Label>
                        <Select>
                            <SelectTrigger>
                                <SelectValue placeholder="Select destination" />
                            </SelectTrigger>
                            <SelectContent>
                                <SelectItem value="placeholder"
                                    >Select destination...</SelectItem
                                >
                            </SelectContent>
                        </Select>
                    </div>
                </div>

                <!-- Find Route Button -->
                <div class="mt-6 flex justify-end">
                    <Button variant="default"> Find Route </Button>
                </div>
            </Card>
        </div>
    </div>
</template>
