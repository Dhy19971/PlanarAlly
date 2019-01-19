<template>
  <SelectContext ref="selectcontext"></SelectContext>
</template>

<script lang="ts">
import SelectContext from "@/game/ui/tools/selectcontext.vue";
import Tool from "@/game/ui/tools/tool.vue";
import SelectionInfo from "../selection/selection_info.vue";

import { getRef } from "@/core/utils";
import { socket } from "@/game/api/socket";
import { GlobalPoint, LocalPoint, Ray, Vector } from "@/game/geom";
import { layerManager } from "@/game/layers/manager";
import { Rect } from "@/game/shapes/rect";
import { gameStore } from "@/game/store";
import { calculateDelta } from "@/game/ui/tools/utils";
import { g2l, g2lx, g2ly, l2g } from "@/game/units";
import { getMouse } from "@/game/utils";
import Component from "vue-class-component";

export enum SelectOperations {
    Noop,
    Resize,
    Drag,
    GroupSelect,
}

const start = new GlobalPoint(-1000, -1000);

@Component({
    components: {
        SelectContext,
    },
})
export default class SelectTool extends Tool {
    name = "Select";
    showContextMenu = false;
    active = false;

    mode = SelectOperations.Noop;
    resizeDirection = "";
    deltaChanged = false;
    // Because we never drag from the asset's (0, 0) coord and want a smoother drag experience
    // we keep track of the actual offset within the asset.
    dragRay = new Ray<LocalPoint>(new LocalPoint(0, 0), new Vector(0, 0));
    selectionStartPoint = start;
    selectionHelper = new Rect(start, 0, 0);
    created() {
        this.selectionHelper.globalCompositeOperation = "source-over";
    }
    onMouseDown(event: MouseEvent) {
        const layer = layerManager.getLayer();
        if (layer === undefined) {
            console.log("No active layer!");
            return;
        }

        if (!this.selectionHelper.owners.includes(gameStore.username)) {
            this.selectionHelper.owners.push(gameStore.username);
        }

        const mouse = getMouse(event);
        const globalMouse = l2g(mouse);

        let hit = false;
        // The selectionStack allows for lower positioned objects that are selected to have precedence during overlap.
        let selectionStack;
        if (!layer.selection.length) selectionStack = layer.shapes;
        else selectionStack = layer.shapes.concat(layer.selection);
        for (let i = selectionStack.length - 1; i >= 0; i--) {
            const shape = selectionStack[i];

            if (!shape.ownedBy()) continue;

            const corner = shape.getBoundingBox().getCorner(globalMouse);

            // Resize case, a corner is selected
            if (corner !== undefined) {
                layer.selection = [shape];
                getRef<SelectionInfo>("selectionInfo").shape = shape;
                this.mode = SelectOperations.Resize;
                this.resizeDirection = corner;
                layer.invalidate(true);
                hit = true;
                break;

                // Drag case, a shape is selected
            } else if (shape.contains(globalMouse)) {
                const selection = shape;
                if (layer.selection.indexOf(selection) === -1) {
                    layer.selection = [selection];
                    getRef<SelectionInfo>("selectionInfo").shape = selection;
                }
                this.mode = SelectOperations.Drag;
                const localRefPoint = g2l(selection.refPoint);
                this.dragRay = new Ray<LocalPoint>(localRefPoint, mouse.subtract(localRefPoint));
                layer.invalidate(true);
                hit = true;
                break;
            }
        }

        // GroupSelect case, draw a selection box to select multiple shapes
        if (!hit) {
            this.mode = SelectOperations.GroupSelect;
            for (const selection of layer.selection) getRef<SelectionInfo>("selectionInfo").shape = selection;

            this.selectionStartPoint = globalMouse;

            this.selectionHelper.refPoint = this.selectionStartPoint;
            this.selectionHelper.w = 0;
            this.selectionHelper.h = 0;

            layer.selection = [this.selectionHelper];
            layer.invalidate(true);
        }
        this.active = true;
    }
    onMouseMove(event: MouseEvent) {
        // if (!this.active) return;   we require mousemove for the resize cursor
        const layer = layerManager.getLayer();
        if (layer === undefined) {
            console.log("No active layer!");
            return;
        }
        const mouse = getMouse(event);
        const globalMouse = l2g(mouse);
        this.deltaChanged = false;

        if (this.mode === SelectOperations.GroupSelect) {
            // Currently draw on active layer
            const endPoint = globalMouse;

            this.selectionHelper.w = Math.abs(endPoint.x - this.selectionStartPoint.x);
            this.selectionHelper.h = Math.abs(endPoint.y - this.selectionStartPoint.y);
            this.selectionHelper.refPoint = new GlobalPoint(
                Math.min(this.selectionStartPoint.x, endPoint.x),
                Math.min(this.selectionStartPoint.y, endPoint.y),
            );
            layer.invalidate(true);
        } else if (layer.selection.length) {
            const og = g2l(layer.selection[layer.selection.length - 1].refPoint);
            const origin = og.add(this.dragRay.direction);
            let delta = mouse.subtract(origin).multiply(1 / gameStore.zoomFactor);
            const ogDelta = delta;
            if (this.mode === SelectOperations.Drag) {
                // If we are on the tokens layer do a movement block check.
                if (layer.name === "tokens" && !(event.shiftKey && gameStore.IS_DM)) {
                    for (const sel of layer.selection) {
                        if (sel.uuid === this.selectionHelper.uuid) continue; // the selection helper should not be treated as a real shape.
                        delta = calculateDelta(delta, sel);
                        if (delta !== ogDelta) this.deltaChanged = true;
                    }
                }
                // Actually apply the delta on all shapes
                for (const sel of layer.selection) {
                    sel.refPoint = sel.refPoint.add(delta);
                    if (sel !== this.selectionHelper) {
                        if (sel.visionObstruction) gameStore.recalculateBV(true);
                        socket.emit("Shape.Update", { shape: sel.asDict(), redraw: true, temporary: true });
                    }
                }
                layer.invalidate(false);
            } else if (this.mode === SelectOperations.Resize) {
                for (const sel of layer.selection) {
                    sel.resize(this.resizeDirection, mouse);
                    if (sel !== this.selectionHelper) {
                        if (sel.visionObstruction) gameStore.recalculateBV(true);
                        socket.emit("Shape.Update", { shape: sel.asDict(), redraw: true, temporary: true });
                    }
                    layer.invalidate(false);
                }
            } else {
                for (const sel of layer.selection) {
                    const bb = sel.getBoundingBox();
                    const gm = globalMouse;
                    if (bb.inCorner(gm, "nw")) {
                        document.body.style.cursor = "nw-resize";
                    } else if (bb.inCorner(gm, "ne")) {
                        document.body.style.cursor = "ne-resize";
                    } else if (bb.inCorner(gm, "se")) {
                        document.body.style.cursor = "se-resize";
                    } else if (bb.inCorner(gm, "sw")) {
                        document.body.style.cursor = "sw-resize";
                    } else {
                        document.body.style.cursor = "default";
                    }
                }
            }
        } else {
            document.body.style.cursor = "default";
        }
    }
    onMouseUp(e: MouseEvent): void {
        if (!this.active) return;
        if (layerManager.getLayer() === undefined) {
            console.log("No active layer!");
            return;
        }
        const layer = layerManager.getLayer()!;

        if (this.mode === SelectOperations.GroupSelect) {
            layer.clearSelection();
            layer.shapes.forEach(shape => {
                if (shape === this.selectionHelper) return;
                const bbox = shape.getBoundingBox();
                if (!shape.ownedBy()) return;
                if (
                    this.selectionHelper!.refPoint.x <= bbox.topRight.x &&
                    this.selectionHelper!.refPoint.x + this.selectionHelper!.w >= bbox.topLeft.x &&
                    this.selectionHelper!.refPoint.y <= bbox.botLeft.y &&
                    this.selectionHelper!.refPoint.y + this.selectionHelper!.h >= bbox.topLeft.y
                ) {
                    layer.selection.push(shape);
                }
            });

            // Push the selection helper as the last element of the selection
            // This makes sure that it will be the first one to be hit in the hit detection onMouseDown
            if (layer.selection.length > 0) layer.selection.push(this.selectionHelper);

            layer.invalidate(true);
        } else if (layer.selection.length) {
            layer.selection.forEach(sel => {
                if (this.mode === SelectOperations.Drag) {
                    if (
                        this.dragRay.origin!.x === g2lx(sel.refPoint.x) &&
                        this.dragRay.origin!.y === g2ly(sel.refPoint.y)
                    )
                        return;

                    if (gameStore.useGrid && !e.altKey && !this.deltaChanged) {
                        sel.snapToGrid();
                    }

                    if (sel !== this.selectionHelper) {
                        if (sel.visionObstruction) gameStore.recalculateBV();
                        socket.emit("Shape.Update", { shape: sel.asDict(), redraw: true, temporary: false });
                    }
                    layer.invalidate(false);
                }
                if (this.mode === SelectOperations.Resize) {
                    if (gameStore.useGrid && !e.altKey) {
                        sel.resizeToGrid();
                    }
                    if (sel !== this.selectionHelper) {
                        if (sel.visionObstruction) gameStore.recalculateBV();
                        socket.emit("Shape.Update", { shape: sel.asDict(), redraw: true, temporary: false });
                    }
                    layer.invalidate(false);
                }
            });
        }
        this.mode = SelectOperations.Noop;
        this.active = false;
    }
    onContextMenu(event: MouseEvent) {
        if (layerManager.getLayer() === undefined) {
            console.log("No active layer!");
            return;
        }
        const layer = layerManager.getLayer()!;
        const mouse = getMouse(event);
        const globalMouse = l2g(mouse);

        for (const shape of layer.selection) {
            if (shape.contains(globalMouse) && shape !== this.selectionHelper) {
                layer.selection = [shape];
                getRef<SelectionInfo>("selectionInfo").shape = shape;
                layer.invalidate(true);
                (<any>this.$parent.$refs.shapecontext).open(event, shape);
                return;
            }
        }
        (<any>this.$refs.selectcontext).open(event);
    }
}
</script>