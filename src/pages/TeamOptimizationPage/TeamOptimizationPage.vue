<template>
    <div>
        <el-breadcrumb>
            <el-breadcrumb-item>Team Calculator</el-breadcrumb-item>
        </el-breadcrumb>
        <el-divider></el-divider>

        <el-drawer
            title="Panel"
            :visible.sync="showAttributeDrawer"
            class="panel"
        >
            <template v-if="!wasmAttribute">
                <el-empty>In theory, you should not see this</el-empty>
            </template>
            <template v-else>
                <div style="padding: 0 20px">
                    <attribute-panel
                        :attribute="wasmAttribute"
                    ></attribute-panel>
                </div>
            </template>
        </el-drawer>

        <el-row :gutter="16">
            <el-col
                :span="6"
                ref="content"
                :style="{ height: contentHeight }"
                class="mona-scroll"
            >
                <div>
                    <my-button1
                        icon="el-icon-caret-right"
                        title="Start Calculation"
                        @click="handleClickStart"
                    ></my-button1>
                    <my-button1
                        icon="el-icon-plus"
                        title="Add Teammate"
                        @click="handleClickAddMember"
                    ></my-button1>
                </div>

                <div
                    v-for="(presetName, index) in presetNames"
                    :key="index"
                    class="member-item"
                >
                    <div style="display: flex; justify-content: space-between; align-items: center">
                        <p class="team-title">Member {{ index + 1 }}</p>
                        <div>
                            <el-button
                                circle
                                size="mini"
                                type="text"
                                icon="el-icon-delete"
                                @click="handleDeleteMember(index)"
                            ></el-button>
                        </div>
                    </div>


                    <p class="common-title2">Character Presets</p>
                    <select-preset
                        v-model="presetNames[index]"
                    ></select-preset>

                    <p class="common-title2">Weights</p>
                    <el-slider
                        v-model="weights[index]"
                        :min="0"
                        :max="1"
                        :step="0.01"
                        :show-input="true"
                        style="padding-left: 8px"
                    ></el-slider>

                    <el-divider v-if="index < presetNames.length - 1"></el-divider>
                </div>
            </el-col>

            <el-col
                :span="18"
                :style="{ height: contentHeight }"
                class="mona-scroll"
            >
                <template v-if="currentResultEntry">
                    <el-input-number
                        :value="resultIndex + 1"
                        @input="handleChangeResultIndex"
                        :min="1"
                        :max="results.length"
                        size="small"
                        style="margin-bottom: 12px"
                    ></el-input-number>
                    <div
                        v-for="(presetName, index) in currentResultEntry"
                        :key="index"
                        class="result-item"
                    >
                        <div class="result-item-top">
                            <div>
<!--                                <span class="result-item-title">{{ characterChs[index] }}</span>-->
                                <el-button
                                    icon="el-icon-s-data"
                                    circle
                                    size="mini"
                                    type="text"
                                    title="View Panel"
                                    @click="handleClickDisplayAttributePanel(index)"
                                ></el-button>
                            </div>

                            <div class="result-item-buttons">
<!--                                <el-button-->
<!--                                    icon="el-icon-plus"-->
<!--                                    circle-->
<!--                                    size="mini"-->
<!--                                    type="text"-->
<!--                                    title="存为套装"-->
<!--                                ></el-button>-->
                            </div>
                        </div>
                        <div class="result-item-content">
                            <artifact-display
                                v-for="artifactId in currentResultEntry[index]"
                                :key="artifactId"
                                :item="artifactsById[artifactId]"
                                :buttons="true"
                                :lock-button="true"
                                :delete-button="false"
                                :edit-button="false"
                                @toggle="handleToggleArtifact(artifactId)"
                            ></artifact-display>
                        </div>
                    </div>
                </template>
                <template v-else>
                    <div
                        style="display: flex; justify-content: center;"
                    >
                        <el-empty></el-empty>
                    </div>
                </template>
            </el-col>
        </el-row>
    </div>
</template>

<script>
import {mapGetters} from "vuex"

import {convertArtifact} from "@util/converter"
import {team_optimize, wasmGetAttribute} from "@/wasm"
import {convertPresetToWasmInterface, getPresetEntryByName} from "@util/preset"
import { newKumiWithArtifacts } from "@util/kumi"
import {toggleArtifact} from "@util/artifacts"
import { deepCopy } from "@util/common"

import SelectCharacter from "@c/select/SelectCharacter"
import SelectWeapon from "@c/select/SelectWeapon"
import ItemConfig from "@c/config/ItemConfig"
import ArtifactDisplay from "@c/display/ArtifactDisplay"
import MyButton1 from "@c/button/MyButton1"
import PresetItem from "@c/display/PresetItem"
import SelectPreset from "@c/select/SelectPreset"
import AttributePanel from "@c/display/AttributePanel"

export default {
    name: "TeamOptimizationPage",
    components: {
        SelectCharacter,
        SelectWeapon,
        ItemConfig,
        ArtifactDisplay,
        MyButton1,
        PresetItem,
        SelectPreset,
        AttributePanel,
    },
    data() {
        return {
            results: [],    // a 3d array
            resultIndex: 0,

            contentHeight: "",

            presetNames: [null],
            weights: [],

            showAttributeDrawer: false,
            wasmAttribute: null,
        }
    },
    mounted() {
        const component = this.$refs["content"]

        this.$nextTick(() => {
            const rect = component.$el.getBoundingClientRect()
            // console.log(rect)
            this.contentHeight = `calc(100vh - ${rect.top}px)`
        })
    },
    methods: {
        handleClickAddMember() {
            if (this.presetNames.length === 8) {
                this.$message.error("Supports up to 8 Teammates")
                return
            }
            this.presetNames.push(null)
            this.weights.push(0)
        },

        handleDeleteMember(index) {
            if (this.presetNames.length === 1) {
                return
            }
            this.$delete(this.presetNames, index)
            this.$delete(this.weights, index)
        },

        handleClickStart() {
            const canStart = this.presets.length === this.presetNames.length
            if (!canStart) {
                this.$message.error("Please select a Character Preset")
                return
            }

            const interfaceWasm = this.optimizeTeamWasmInterface
            const artifacts = this.filteredArtifactsWasm
            // console.log(artifacts)
            // console.log(interfaceWasm)

            const loading = this.$loading({
                lock: true,
                text: "Doing Some Magic Please Wait",
                background: "rgba(0, 0, 0, 0.7)",
            })

            team_optimize(interfaceWasm, artifacts).then(result => {
                // console.log(result)
                this.results = result.artifacts
                this.resultIndex = 0
            }).catch(e => {
                console.log(e)
            }).finally(() => {
                loading.close()
            })
        },

        handleChangeResultIndex(index) {
            this.resultIndex = index - 1
        },

        handleClickDisplayAttributePanel: async function (index) {
            const input = this.wasmGetAttributeInterface(index)
            // console.log(input)
            const result = await wasmGetAttribute(input)
            this.wasmAttribute = result

            this.showAttributeDrawer = true
            // console.log(result)
        },

        wasmGetAttributeInterface(index) {
            let artifacts = []
            if (this.currentResultEntry) {
                const artifactIds = Object.values(this.currentResultEntry[index])
                const artifactsOldFormat = artifactIds.map(x => this.artifactsById[x]).filter(x => x)
                artifacts = artifactsOldFormat.map(a => convertArtifact(a))
            }
            // console.log(this.presets[index])

            return {
                character: this.presets[index].item.character,
                weapon: this.presets[index].item.weapon,
                buffs: this.presets[index].item.buffs,
                artifacts,
            }
        },

        // not used, todo
        handleClickSaveAsKumi(index) {
            let artifacts = []
            if (this.currentResultEntry) {
                const artifactIds = Object.values(this.currentResultEntry[index])
                artifacts = artifactIds.map(x => this.artifactsById[x]).filter(x => x)
            }
        },

        handleToggleArtifact(artifactId) {
            toggleArtifact(artifactId)
        }
    },
    computed: {
        ...mapGetters("artifacts", {
            artifactsFlat: "allFlat",
            artifactsById: "artifactsById",
        }),

        singleInterfaces() {
            return this.presets.map(x => convertPresetToWasmInterface(x.item))
        },

        currentResultEntry() {
            if (this.results.length === 0) {
                return null
            } else {
                return this.results[this.resultIndex]
            }
        },

        filteredArtifacts() {
            let results = []
            for (let artifact of this.artifactsFlat) {
                if (artifact.level >= 16) {
                    results.push(artifact)
                }
            }
            return results.filter(a => !a.omit)
        },

        filteredArtifactsWasm() {
            return this.filteredArtifacts.map(convertArtifact)
        },

        presets() {
            let results = []
            for (let name of this.presetNames) {
                if (name) {
                    results.push(getPresetEntryByName(name))
                }
            }
            return results
        },

        optimizeTeamHyperParamInterface() {
            // todo
            return {
                mva_step: 5,
                work_space: 1000,
                max_re_optimize: 5,
                max_search: 1000000,
                count: 1000,
            }
        },


        optimizeTeamWasmInterface() {
            // sort by weight
            // let temp = []
            // for (let i = 0; i < this.singleInterfaces.length; i++) {
            //     temp.push([this.singleInterfaces[i], this.weights[i]])
            // }
            // temp.sort((a, b) => b[1] - a[1])
            //
            // const interfaces = temp.map(x => x[0])
            // const weights = temp.map(x => x[1])
            return {
                // single_interfaces: interfaces,
                // weights: weights,
                single_interfaces: this.singleInterfaces,
                weights: this.weights,
                hyper_param: this.optimizeTeamHyperParamInterface
            }
        }
    },
}
</script>

<style scoped lang="scss">

.member-item {
    margin-bottom: 16px;
    //box-shadow: 0 0 10px 1px #00000011;
    //padding: 12px;

    &:last-of-type {
        margin-bottom: 64px;
    }

    .title {
        font-size: 14px;
        margin: 0;
        margin-bottom: 12px;
    }

    .image {
        width: 64px;
        height: 64px;
        display: inline-block;
        border-radius: 50%;
        margin-top: 12px;
    }
}

.common-title2 {
    font-size: 12px;
    color: #ffff;
}

.team-title {
    font-size: 0.9rem;
    font-weight: bold;
    color: #ffff;
    //border-left: 2px solid #409EFF;
    //padding-left: 12px;
}

.result-item {
    margin-bottom: 12px;

    &:last-of-type {
        margin-bottom: 0;
    }

    .result-item-top {
        display: flex;
        align-items: center;
        justify-content: space-between;
        height: 36px;
        border-bottom: 1px solid #00000011;

        .result-item-title {
            font-size: 12px;
        }

        .result-item-buttons {
            display: flex;
            align-items: center;
        }
    }

    .result-item-content {
        padding-top: 12px;
        display: flex;
        flex-wrap: wrap;
        gap: 12px;
    }
}
</style>