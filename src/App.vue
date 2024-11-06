<script setup lang="ts">
import { Card, CardHeader, CardTitle, CardDescription } from '@/components/ui/card'
import { Label } from '@/components/ui/label'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
import { Button } from '@/components/ui/button'
</script>

<template>
  <div class="min-h-screen bg-slate-50 p-8">
    <!-- Main Container -->
    <div class="max-w-7xl mx-auto space-y-6">
      <!-- Header Card -->
      <Card class="w-full">
        <CardHeader>
          <CardTitle>School Navigation System</CardTitle>
          <CardDescription>Find the best route between locations</CardDescription>
        </CardHeader>
      </Card>

      <!-- Map Container Card -->
      <Card class="w-full"> </Card>

      <!-- Navigation Controls Card -->
      <Card class="w-full p-6">
        <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
          <!-- Algorithm Selection -->
          <div class="space-y-2">
            <Label class="text-md font-medium" for="algorithm"> Algorithm </Label>
            <Select>
              <SelectTrigger>
                <SelectValue placeholder="Select algorithm" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="dijkstra">Dijkstra's Algorithm</SelectItem>
                <SelectItem value="bfs">Breadth First Search</SelectItem>
                <SelectItem value="dfs">Depth First Search</SelectItem>
              </SelectContent>
            </Select>
          </div>

          <!-- Start Location -->
          <div class="space-y-2">
            <Label for="start">Start Location</Label>
            <Select>
              <SelectTrigger>
                <SelectValue placeholder="Select start location" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="placeholder">Select start location...</SelectItem>
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
                <SelectItem value="placeholder">Select destination...</SelectItem>
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
