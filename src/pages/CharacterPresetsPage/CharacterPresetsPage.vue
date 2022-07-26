<template>
    <div>
        <el-breadcrumb>
            <el-breadcrumb-item>Calculation Presets</el-breadcrumb-item>
        </el-breadcrumb>
        <el-divider></el-divider>

        <el-dialog
            :visible.sync="showImportDialog"
            title="Import"
            width="60%"
        >
            <import-block ref="importBlock"></import-block>

            <template #footer>
                <el-button @click="showImportDialog = false">No</el-button>
                <el-button type="primary" @click="handleImport">Yes</el-button>
            </template>
        </el-dialog>

        <el-drawer
            :visible.sync="showArtifactsScoreDrawer"
            title="Recommended Artifact"
        >
            <el-empty v-if="artifactScoreList.length === 0"></el-empty>
            <div v-else class="artifact-score-list-div">
                <div
                    v-for="(item, index) in artifactsScoreListCut"
                    :key="index"
                >
                    <artifact-display
                        :item="artifactsById[item[0]]"
                        :show-back="true"
                        :back-value="item[1]"
                        class="artifact-score-item"
                    ></artifact-display>
                </div>
            </div>
        </el-drawer>

        <div class="toolbar">
            <el-button
                icon="el-icon-download"
                size="mini"
                @click="handleExportAll"
            >Export All</el-button>
            <el-button
                icon="el-icon-upload2"
                size="mini"
                @click="handleClickImport"
            >Import</el-button>
        </div>

        <div class="body" v-if="presetsLength > 0">
            <preset-item
                v-for="entry in allFlat"
                :item="entry.item"
                :name="entry.name"
                @delete="handleDeletePreset(entry.name)"
                @download="handleDownload(entry.name)"
                class="item"
                @click="test(entry.name)"
            ></preset-item>
        </div>
        <template v-else>

            <el-empty>
                <p
                    style="font-size: 0.9em; color: #ffff"
                >
                    Go to<span class="route-item" @click="$router.replace('/calculate')"> Character Calculator</span>And Add Calculation Presets
                </p>
            </el-empty>
        </template>
    </div>
</template>

<script>
import { mapGetters } from "vuex"

import { deletePreset, getPresetEntryByName, checkImportFormat, createOrUpdatePreset, upgradePresetItem } from "@util/preset"
import { downloadString } from "@util/common"
import { convertArtifact } from "@util/converter"
import { wasmGetArtifactsRankByCharacter } from "@/wasm"

import PresetItem from "@c/display/PresetItem"
import ImportBlock from "@c/misc/ImportBlock"
import ArtifactDisplay from "@c/display/ArtifactDisplay"

export default {
    name: "CharacterPresetsPage",
    components: {
        PresetItem,
        ImportBlock,
        ArtifactDisplay,
    },
    created() {
    },
    data() {
        return {
            showImportDialog: false,
            showArtifactsScoreDrawer: false,

            artifactScoreList: [],
        }
    },
    methods: {
        checkImportType(obj) {
            if (Object.prototype.hasOwnProperty.call(obj, "data")) {
                return "multi";
            } else {
                return "single"
            }
        },

        async handleImport() {
            const component = this.$refs.importBlock
            if (!component) {
                return
            }

            const text = await component.getReadPromise()

            let obj = null
            try {
                obj = JSON.parse(text)
            } catch {
                obj = null
            }

            const checkObj = checkImportFormat(obj)

            if (!checkObj) {
                this.$message.error("Import Format Error")
            } else {
                for (let entry of obj) {
                    createOrUpdatePreset(entry.item, entry.name)
                }

                this.showImportDialog = false
            }
        },

        handleDeletePreset(name) {
            deletePreset(name)
        },

        handleDownload(name) {
            const entry = getPresetEntryByName(name)
            const temp = [entry]
            const str = JSON.stringify(temp)

            downloadString(str, "text/plain", name)
        },

        handleExportAll() {
            const str = JSON.stringify(this.allFlat)
            
            downloadString(str, "text/plain", "Preset")
        },

        handleClickImport() {
            this.showImportDialog = true
        },

        test(name) {
            console.log(name)

            const item = getPresetEntryByName(name).item
            if (!item) {
                return
            }

            const characterInterface = item.character
            const weaponInterface = item.weapon
            const tfInterface = item.targetFunction

            const artifacts = this.allArtifactsFlat.map(a => convertArtifact(a))

            wasmGetArtifactsRankByCharacter(characterInterface, weaponInterface, tfInterface, artifacts).then(result => {
                result = result.filter(item => {
                    return this.artifactsById[item[0]].level === 0
                })

                this.artifactScoreList = result

                const maxValue = result.map(x => x[1]).reduce((p, c) => Math.max(p, c), 0)
                console.log(maxValue)
                if (maxValue > 0) {
                    for (let i = 0; i < result.length; i++) {
                        result[i][1] /= maxValue
                    }
                }


                this.showArtifactsScoreDrawer = true
            })
        }
    },
    computed: {
        ...mapGetters("presets", ["allFlat"]),
        ...mapGetters("artifacts", {
            "allArtifactsFlat": "allFlat",
            "artifactsById": "artifactsById"
        }),

        presetsLength() {
            return this.allFlat.length
        },

        artifactsScoreListCut() {
            return this.artifactScoreList.slice(0, 20)
        },
    }
}
</script>

<style scoped lang="scss">

.item {
    margin: 0 16px 16px 0;
   
}

.body {
    display: flex;
    flex-wrap: wrap;
    
}

.toolbar {
    margin-bottom: 20px;
    
}

.artifact-score-list-div {
    padding: 0 20px;

    .artifact-score-item {
        width: 100%;
        margin-bottom: 12px;
    }
}

.route-item {
    padding: 4px;
    transition: 200ms;
    border-radius: 2px;
    cursor: pointer;
    color: rgb(86,69,137);

    &:hover {
        background-color: rgb(86,69,137);
        color: white;
    }
}
</style>