<template>
    <footer class="absolute bottom-0 left-12 right-0 p-2 z-50">
        <LightOrDark>
            <template #dark>
                <img src="/zivergeLogo.png" class="h-5" />
            </template>
            <template #light>
                <img src="/zivergeLogo.png" class="h-5" />
            </template>
        </LightOrDark>
    </footer>

    <footer class="absolute bottom-0 right-12 p-2 z-50">
        <SlideCurrentNo /> /
        <SlidesTotal />
    </footer>
</template>