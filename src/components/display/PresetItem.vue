<template>
    <div class="item br-3" @click="$emit('click')">
        <div class="header">
            <span class="fs-12">{{ name }}</span>
            <div v-if="toolbar" class="buttons flex-row">
                <el-button
                    icon="el-icon-delete"
                    type="text"
                    size="mini"
                    circle
                    @click.stop="$emit('delete')"
                    class="button"
                    title="Delete"
                ></el-button>
                <el-button
                    icon="el-icon-download"
                    type="text"
                    size="mini"
                    circle
                    @click.stop="$emit('download')"
                    class="button"
                    title="导出"
                ></el-button>
            </div>
        </div>
        <div class="body">
            <div class="detail-div fs-12">
                <img :src="characterAvatar" class="c-avatar br-50p">
                <span>{{ characterChs }}</span>
            </div>
            <div class="detail-div fs-12">
                <img :src="weaponData.url" class="w-avatar br-50p">
                <span>{{ weaponData.chs }}</span>
            </div>
            <div class="detail-div fs-12">
                <img :src="tfData.badge" class="tf-avatar br-50p">
                <span>{{ tfData.chs }}</span>
            </div>
        </div>
    </div>
</template>

<script>
import { characterData } from "@character"
import { weaponData } from "@weapon"
import { targetFunctionData } from "@targetFunction"

export default {
    name: "PresetItem",
    props: {
        item: {
            type: Object,
            required: true,
        },
        name: {},
        toolbar: {
            type: Boolean,
            default: true,
        }
    },
    computed: {
        characterName() {
            return this.item.character.name
        },

        characterAvatar() {
            const data = characterData[this.characterName]
            return data.avatar
        },

        characterChs() {
            const data = characterData[this.characterName]
            return data.chs
        },

        weaponData() {
            return weaponData[this.item.weapon.name]
        },

        tfData() {
            return targetFunctionData[this.item.targetFunction.name]
        }
    }

}
</script>

<style lang="scss" scoped>
.item {
    //box-shadow: 0 0 10px 1px #00000011;
    border: 1px solid #00000011;
    display: inline-block;
    transition: 300ms;

    &:hover {
        background-color: #00000008;
        cursor: pointer;
    }

    .header {
        height: 32px;
        border-bottom: 1px solid #00000011;

        span {
            line-height: 32px;
            padding: 4px 8px;
            color: #3cafe4;
        }

        .buttons {
            float: right;
            height: 32px;

            .button {
                margin: 0;
            }
        }
    }

    .body {
        padding: 8px;
        display: flex;

        .detail-div {
            margin-right: 12px;

            span {
                width: 64px;
                text-align: center;
                display: block;
                padding-top: 8px;
            }

            &:last-of-type {
                margin: 0;
            }
        }
    }
    
}

.c-avatar, .w-avatar, .tf-avatar {
    display: block;
    height: 64px;
    width: 64px;
    border: 1px solid #e9e9e9;
}
</style>