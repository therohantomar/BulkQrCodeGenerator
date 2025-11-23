<template>
    <main
        class="relative w-full min-h-screen flex flex-col justify-center py-2 overflow-hidden bg-[#0e0e1a] px-6"
    >
        <div
            class="absolute top-10 left-16 w-72 h-72 bg-pink-500/30 blur-[120px] rounded-full"
        ></div>
        <div
            class="absolute bottom-10 right-16 w-72 h-72 bg-purple-500/30 blur-[130px] rounded-full"
        ></div>
        <div
            class="absolute top-1/2 right-1/3 w-64 h-64 bg-blue-500/25 blur-[110px] rounded-full"
        ></div>

        <Header />
        <Upload />
    </main>
</template>
<script>
import Header from "./components/Header.vue";
import Upload from "./components/Upload.vue";

export default {
    components: {
        Header,
        Upload,
    },
};
</script>
