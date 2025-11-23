<template>
    <div
        class="max-w-3xl mx-auto mt-12 p-10 bg-white/10 backdrop-blur-2xl border border-white/20 shadow-[0_8px_40px_rgba(0,0,0,0.25)] rounded-3xl"
    >
        <h2
            class="text-4xl font-extrabold text-white mb-8 text-center font-jakarta tracking-tight drop-shadow"
        >
            Upload CSV File
        </h2>

        <!-- File Input -->
        <div class="flex flex-col items-center gap-5">
            <label
                for="csvInput"
                class="w-full cursor-pointer bg-gradient-to-r from-indigo-500 via-purple-500 to-pink-500 text-white py-3 rounded-2xl text-center text-lg font-semibold shadow-xl hover:scale-[1.02] active:scale-[0.98] transition-all"
            >
                Choose CSV File
            </label>

            <input
                id="csvInput"
                type="file"
                accept=".csv"
                class="hidden"
                @change="handleFile"
            />

            <p v-if="fileName" class="text-white/80 text-sm font-medium">
                Selected: {{ fileName }}
            </p>

            <button
                @click="downloadExample"
                class="mt-1 px-5 py-2.5 bg-emerald-600 text-white rounded-xl shadow hover:bg-emerald-700 hover:scale-105 transition text-sm"
            >
                Download Example CSV
            </button>
        </div>

        <!-- Color Inputs -->
        <div class="grid grid-cols-2 gap-5 mt-6">
            <div>
                <label class="text-white/80 text-sm">QR Dot Color</label>
                <input
                    v-model="dotColor"
                    placeholder="#000000"
                    class="w-full mt-1 px-3 py-2 rounded-xl bg-white/10 text-white border border-white/20 outline-none"
                />
            </div>

            <div>
                <label class="text-white/80 text-sm">Corner Color</label>
                <input
                    v-model="cornerColor"
                    placeholder="#000000"
                    class="w-full mt-1 px-3 py-2 rounded-xl bg-white/10 text-white border border-white/20 outline-none"
                />
            </div>
        </div>

        <!-- Generate Button -->
        <button
            @click="generate"
            :disabled="!csvData"
            class="w-full mt-8 bg-blue-700 disabled:bg-gray-600 disabled:cursor-not-allowed text-white py-4 rounded-2xl text-xl font-bold shadow-xl hover:bg-blue-800 hover:scale-[1.01] transition"
        >
            Generate Fancy QR ZIP
        </button>

        <p
            v-if="loading"
            class="text-center mt-5 text-white/90 font-medium text-lg animate-pulse"
        >
            Processing… Please wait
        </p>
    </div>

    <!-- HEADLESS UI MODAL -->
    <TransitionRoot appear :show="previewPopup !== null" as="template">
        <Dialog as="div" class="relative z-50" @close="previewPopup = null">
            <TransitionChild
                as="template"
                enter="duration-300 ease-out"
                enter-from="opacity-0"
                enter-to="opacity-100"
                leave="duration-200 ease-in"
                leave-from="opacity-100"
                leave-to="opacity-0"
            >
                <div class="fixed inset-0 bg-black/60 backdrop-blur-md" />
            </TransitionChild>

            <div class="fixed inset-0 overflow-y-auto">
                <div class="flex min-h-full items-center justify-center p-4">
                    <TransitionChild
                        as="template"
                        enter="duration-300 ease-out"
                        enter-from="opacity-0 scale-95"
                        enter-to="opacity-100 scale-100"
                        leave="duration-200 ease-in"
                        leave-from="opacity-100 scale-100"
                        leave-to="opacity-0 scale-95"
                    >
                        <DialogPanel
                            class="w-full max-w-md transform overflow-hidden rounded-3xl bg-white/20 backdrop-blur-2xl p-6 text-center shadow-xl transition-all border border-white/30"
                        >
                            <DialogTitle class="text-white text-lg font-medium">
                                QR Preview
                            </DialogTitle>

                            <img
                                :src="previewPopup"
                                class="w-60 h-60 mx-auto mt-4 rounded-xl shadow-xl"
                            />

                            <button
                                @click="previewPopup = null"
                                class="mt-6 px-5 py-2 bg-red-500 text-white rounded-xl hover:bg-red-600 transition"
                            >
                                Close
                            </button>
                        </DialogPanel>
                    </TransitionChild>
                </div>
            </div>
        </Dialog>
    </TransitionRoot>

    <!-- SMALL THUMBNAILS -->
    <div
        v-if="preview.length"
        class="max-w-3xl mx-auto grid grid-cols-3 gap-5 mt-10"
    >
        <div
            v-for="(img, i) in preview"
            :key="i"
            @click="previewPopup = img"
            class="p-3 bg-white/20 backdrop-blur-xl border border-white/20 rounded-2xl shadow hover:scale-110 hover:border-white/40 cursor-pointer transition"
        >
            <img :src="img" class="w-28 h-28 mx-auto rounded-xl shadow" />
            <p class="text-white/80 text-center mt-2 text-sm font-medium">
                Preview {{ i + 1 }}
            </p>
        </div>
    </div>
</template>

<script>
import Papa from "papaparse";
import JSZip from "jszip";
import { saveAs } from "file-saver";
import QRCodeStyling from "qr-code-styling";

import {
    Dialog,
    DialogPanel,
    DialogTitle,
    TransitionChild,
    TransitionRoot,
} from "@headlessui/vue";

export default {
    name: "UploadComponent",

    components: {
        Dialog,
        DialogPanel,
        DialogTitle,
        TransitionChild,
        TransitionRoot,
    },

    data() {
        return {
            fileName: null,
            csvData: null,
            rows: [],
            preview: [],
            previewPopup: null,
            loading: false,

            dotColor: "",
            cornerColor: "",
        };
    },

    methods: {
        downloadExample() {
            const sample = `FirstName,LastName,CompanyName,PhoneNumber
John,Doe,ABC Corp,9876543210
Jane,Smith,XYZ Pvt Ltd,9123456780`;

            const blob = new Blob([sample], { type: "text/csv" });
            saveAs(blob, "example.csv");
        },

        handleFile(e) {
            const file = e.target.files[0];
            if (!file) return;

            this.fileName = file.name;

            const reader = new FileReader();
            reader.onload = (event) => {
                this.csvData = event.target.result;
                this.parseCSV();
            };
            reader.readAsText(file);
        },

        parseCSV() {
            const parsed = Papa.parse(this.csvData, {
                header: true,
                skipEmptyLines: true,
            });

            const required = [
                "FirstName",
                "LastName",
                "CompanyName",
                "PhoneNumber",
            ];
            const headers = Object.keys(parsed.data[0] || {});

            for (const col of required) {
                if (!headers.includes(col)) {
                    alert(`CSV missing required column: ${col}`);
                    this.csvData = null;
                    return;
                }
            }

            this.rows = parsed.data;
        },

        cleanPhone(num) {
            if (!num) return "";
            let cleaned = String(num).replace(/[^0-9]/g, "");
            if (!cleaned.startsWith("91")) cleaned = "91" + cleaned;
            return "+" + cleaned;
        },

        makeVCard(rec) {
            const phone = this.cleanPhone(rec.PhoneNumber);
            return `BEGIN:VCARD
VERSION:3.0
N:${rec.LastName};${rec.FirstName};;;
FN:${rec.FirstName} ${rec.LastName}
ORG:${rec.CompanyName}
TEL;TYPE=CELL:${phone}
END:VCARD`;
        },

        async generateFancyQR(vcard, name) {
            const qr = new QRCodeStyling({
                width: 600,
                height: 600,
                type: "png",
                data: vcard,
                margin: 0,
                qrOptions: { mode: "Byte", errorCorrectionLevel: "H" },
                dotsOptions: {
                    type: "rounded",
                    color: this.dotColor?.trim() || "#000000",
                },
                cornersSquareOptions: {
                    type: "extra-rounded",
                    color: this.cornerColor?.trim() || "#000000",
                },
                cornersDotOptions: {
                    type: "dot",
                    color: this.cornerColor?.trim() || "#000000",
                },
                backgroundOptions: {
                    color: "#ffffff",
                },
            });

            const blob = await qr.getRawData("png");
            return await this.blobToBase64(blob);
        },

        blobToBase64(blob) {
            return new Promise((resolve) => {
                const reader = new FileReader();
                reader.onloadend = () => resolve(reader.result);
                reader.readAsDataURL(blob);
            });
        },

        async generate() {
            if (!this.rows.length) return alert("No CSV data available");

            this.loading = true;
            this.preview = [];

            const zip = new JSZip();

            for (let i = 0; i < this.rows.length; i++) {
                const rec = this.rows[i];
                const name = `${rec.FirstName} ${rec.LastName}`.trim();
                const vcard = this.makeVCard(rec);

                const png = await this.generateFancyQR(vcard, name);

                if (i < 4) this.preview.push(png);

                zip.file(
                    `${i + 1}_${name.replace(/\s+/g, "_")}.png`,
                    png.split(",")[1],
                    { base64: true },
                );
            }

            const blob = await zip.generateAsync({ type: "blob" });
            saveAs(blob, "fancy_qr_codes.zip");
            this.loading = false;
        },
    },
};
</script>
