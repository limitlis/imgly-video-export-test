
<script setup>
import { ref, onMounted } from 'vue';
import { defaults } from 'lodash-es';
const imglyEditorURL = 'https://cdn.img.ly/packages/imgly/cesdk-js/1.43.0/cesdk.umd.js';

const baseURL = `${import.meta.env.VITE_PUBLIC_URL_HOSTNAME}/imgly`;
const publicLicense = import.meta.env.VITE_PUBLIC_LICENSE;
const imglyLibReady = ref(false);
const editorInitialized = ref(false);
let _cesdk;
const defaultConfig = {
    core: {
        // Specify location of core assets, required by the engine.
        baseURL: `${baseURL}/core`
    },
    baseURL,
    ui: {
        elements: {
            navigation: {
                action: {
                    close: true,
                    back: true,
                    save: true,
                    load: false,
                    export: false
                }
            }
        },
        panels: {
            inspector: {
                show: false,
            },
        },
        pageFormats: {
            '1080p': {
                default: true,
                width: 1920,
                height: 1080,
                unit: 'Pixel',
                fixedOrientation: true
            },
        }
    }
};

const init = async () => {
    let scriptPromise;
    if (!document.getElementById('imglyEditor')) {
        console.log('CreativeEditor component embedding the imgly script');
        imglyLibReady.value = false;
        editorInitialized.value = false;

        scriptPromise = new Promise((resolve) => {
            const imglyLib = document.createElement('script');
            imglyLib.src = imglyEditorURL;
            imglyLib.id = 'imglyEditor';
            imglyLib.onload = () => {
                // console.log('imgly editor loaded from CDN');
                resolve();
                imglyLibReady.value = true;
            }
            document.getElementsByTagName('head')[0].appendChild(imglyLib);
        })
    } else {
        if (!imglyLibReady.value) {
            // DX HMR FIx: If the script isn't on the document anymore then clear out these flags too.
            imglyLibReady.value = true;
            editorInitialized.value = false;
        }
        scriptPromise = Promise.resolve();
    }
    return scriptPromise.then(() => {
        console.log('CreativeEditor component begin init');
        let initPromise;
        if (!editorInitialized.value || !_cesdk) {
            let ceConfig = {
                license: publicLicense,
                callbacks: {
                    async onSave() {
                        console.log('Performing Save');
                        const activeScene = _cesdk.engine.scene.get();
                        const currentPage = _cesdk.engine.scene.getCurrentPage();
                        let mimeType = 'video/mp4', options, blob, metadata = {};
                        const progressCallback = (renderedFrames, encodedFrames, totalFrames) => {
                            console.log(
                                'Rendered',
                                renderedFrames,
                                'frames and encoded',
                                encodedFrames,
                                'frames out of',
                                totalFrames
                            );
                        };
                        // Get duration
                        metadata.duration = _cesdk.engine.block.getDuration(currentPage);
                        try {
                            blob = await _cesdk.engine.block.exportVideo(
                                currentPage,
                                mimeType,
                                progressCallback,
                                {}
                            );

                        } catch (error) {
                            console.error(error);
                        }
                    },
                    // Use local uploads: https://img.ly/docs/cesdk/ui/guides/upload-images?language=js#local-uploads
                    onUpload: 'local'
                },
                // mash together defaults from above and settings passed in via $props
                ...defaults(defaultConfig, {})
            };
            // once loaded CreativeEditorSDK is injected globally
            initPromise = CreativeEditorSDK.create('#cesdk_container', ceConfig)

        } else {
            initPromise = new Promise((resolve) => {
                resolve(_cesdk);
            });
        }
        return initPromise.then(async (instance) => {
            console.log('CE editor initialized...', instance);
            const { engine } = instance;
            if (!_cesdk) {
                console.log('CE editor doing initial setup... for _cesdk');
                _cesdk = instance;
                instance.ui.setDockOrder([]);
                // Remove ability to add pages.
                instance.feature.enable('ly.img.page.add', false);
                // Disable placeholder and preview features
                instance.feature.enable('ly.img.placeholder', false);
                instance.feature.enable('ly.img.preview', false);
                instance.feature.enable('ly.img.replace', false);
                // set basePath, so that CE doesn't try and load local assets from their CDN
                // engine.editor.setSettingString('basePath', `${onLocal ? baseURL : '/imgly/'}`);
                engine.editor.setSettingString('basePath', '');
            }
            await engine.scene.createFromVideo(
                // 'https://img.ly/static/ubq_video_samples/bbb.mp4'
                `/example-vid.mov`
            );

            editorInitialized.value = true;
        });
    })
}
onMounted(() => {
    init();
})
</script>

<template>
    <div id="cesdk_container" style="height: 100vh; width: 100%">
        <div v-if="!imglyLibReady"
            style="background: rgb(204, 204, 204);display: flex;height: 100%;width: 100%;justify-content: center;align-items: center;">
            Loading</div>
    </div>
</template>

