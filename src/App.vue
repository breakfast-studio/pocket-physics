<template>
    <Lunchbox
        background="aliceblue"
        :cameraPosition="[4 * 0.6, 3 * 0.6, 10 * 0.6]"
        :cameraLook="[0, 0, 0]"
        shadow
    >
        <!-- lighting -->
        <directionalLight
            castShadow
            :position="[-50, 100, 100]"
            :intensity="0.4"
        />
        <ambientLight :intensity="0.4" />

        <!-- ground -->
        <mesh
            receiveShadow
            :position-y="-0.1"
            :scale="100"
            :rotation-x="Math.PI * -0.5"
            @click="paletteIdx++"
        >
            <planeGeometry />
            <meshToonMaterial color="mistyrose" />
        </mesh>

        <!-- boxes -->
        <group :scale="0.02">
            <mesh
                v-for="(box, i) in points"
                :key="i"
                :position-x="box.cpos.x"
                :position-z="box.cpos.y"
                :scale="5"
                castShadow
                receiveShadow
            >
                <sphereGeometry />
                <meshToonMaterial :color="palette[i % palette.length]" />
            </mesh>
        </group>
    </Lunchbox>
</template>

<script setup lang="ts">
import { onBeforeRender } from 'lunchboxjs'
import { computed, reactive, ref, watch } from 'vue'
import {
    add,
    distance,
    scale,
    sub,
    v2,
    accelerate,
    inertia,
    solveGravitation,
    overlapAABBAABB,
    collisionResponseAABB,
    AABBOverlapResult,
    copy,
} from 'pocket-physics'

// colors
import palettes from 'nice-color-palettes'
const paletteIdx = ref(49)
watch(paletteIdx, console.log)
const palette = computed(() => palettes[paletteIdx.value])

const width = 5
const height = 5
const count = 25

const makeBox = (x: number, y: number) => {
    return {
        id: 'id-' + Math.floor(Math.random() * 10000000),
        cpos: v2(x, y),
        ppos: v2(x, y),
        acel: v2(),
        mass: 10,
        w: 10,
        h: 10,
    }
}

const points = reactive([] as ReturnType<typeof makeBox>[])

for (let c = count, i = 0; i < c; i++) {
    const centerX = width / 2
    const centerY = height / 2
    const distance = 200
    const cos = Math.cos(i)
    const sin = Math.sin(i)
    const x = centerX + cos * distance
    const y = centerY + sin * distance
    points.push(makeBox(x, y))
}

const GRAVITATIONAL_POINT = {
    cpos: v2(width / 2, height / 2),
    ppos: v2(width / 2, height / 2),
    acel: v2(),
    mass: 100000,
}

onBeforeRender(() => {
    for (let i = 0; i < points.length; i++) {
        const point = points[i]
        const dist = distance(point.cpos, GRAVITATIONAL_POINT.cpos)
        dist > 100 &&
            solveGravitation(
                point,
                point.mass,
                GRAVITATIONAL_POINT,
                GRAVITATIONAL_POINT.mass
            )
    }

    const dt = 1
    for (let i = 0; i < points.length; i++) {
        const point = points[i]
        accelerate(point, dt)
    }

    const collisions = []
    const handled = []
    const collision: AABBOverlapResult = {
        resolve: v2(),
        hitPos: v2(),
        normal: v2(),
    }

    for (let i = 0; i < points.length; i++) {
        for (let j = i + 1; j < points.length; j++) {
            const box1 = points[i]
            const box2 = points[j]
            const isOverlapping = overlapAABBAABB(
                box1.cpos.x,
                box1.cpos.y,
                box1.w,
                box1.h,
                box2.cpos.x,
                box2.cpos.y,
                box2.w,
                box2.h,
                collision
            )

            if (
                isOverlapping &&
                handled.indexOf(box1.id + ',' + box2.id) === -1 &&
                handled.indexOf(box2.id + ',' + box1.id) === -1
            ) {
                // move to non-overlapping position
                const overlapHalf = scale(v2(), collision.resolve, 0.5)
                add(box2.cpos, box2.cpos, overlapHalf)
                add(box2.ppos, box2.ppos, overlapHalf)
                sub(box1.cpos, box1.cpos, overlapHalf)
                sub(box1.ppos, box1.ppos, overlapHalf)

                // for debugging
                // render(points, ctx)

                const box1v = v2()
                const box2v = v2()

                const restitution = 1
                const staticFriction = 0.9
                const dynamicFriction = 0.01

                collisionResponseAABB(
                    box1.cpos,
                    box1.ppos,
                    box1.mass,
                    restitution,
                    staticFriction,
                    dynamicFriction,
                    box2.cpos,
                    box2.ppos,
                    box2.mass,
                    restitution,
                    staticFriction,
                    dynamicFriction,
                    // Allow the response function to recompute a normal based on the
                    // axis between the centers of the boxes. this produces a more
                    // natural looking collision.
                    // collision.normal,
                    v2(),
                    box1v,
                    box2v
                )

                // Apply the new velocity
                sub(box1.ppos, box1.cpos, box1v)
                sub(box2.ppos, box2.cpos, box2v)

                // for debugging
                // render(points, ctx)

                handled.push(box1.id + ',' + box2.id)
                handled.push(box2.id + ',' + box1.id)
            }
        }
    }

    for (let i = 0; i < points.length; i++) {
        const point = points[i]
        inertia(point)
    }
})
</script>