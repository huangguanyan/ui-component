<script>
/**
 * Create 2022/05/19
 * Author: NX7353
 * 
 * 车牌号输入法设计思路梳理
 * 1. 值的结构
 * 2. 输入法的布局/key切换
 * 3. 事件处理
 * 4. 变化场景判断
 * 
 * 条件 List:
 * 1. 初始默认空值处理，默认普通车牌选项
 * 2. 手动切换车牌类型：普通车牌、新能源车牌 - 新能源车牌比普通车牌多一位数
 * 3. 自动切换输入内容：车牌首字符为中文；第二位英文；第三至最后一位为英文+字符，英文不能输入O和I；特殊字符在最后一位追加
 */
export default {
    props: {
        licensePlate: {
            type: String,
            default: null
        },
        visible: {
            type: Boolean,
            default: false
        }
    },
    /**
     * @param {number} licensePlateType 0/1 - 0:普通车牌, 1:新能源车牌
     * @param {number} switchKey 0/1 - 切换 keys 显示
     */
    data () {
        return {
            licensePlateType: 0,
            keys: [
                [1, 2, 3, 4, 5, 6, 7, 8, 9, 0],
                [
                    [
                        ['京', '湘', '津', '鄂', '沪', '粤', '渝', '琼', '冀', '川'], 
                        ['晋', '贵', '辽', '云', '吉', '陕', '黑', '甘', '苏'], 
                        ['青', '浙', '皖', '藏', '闽', '蒙', '赣', '桂'], 
                        ['鲁', '宁', '豫', '新']
                    ],
                    [
                        ['Q', 'W', 'E', 'R', 'T', 'Y', 'U', 'P'], 
                        ['A', 'S', 'D', 'F', 'G', 'H', 'J', 'K', 'L'], 
                        ['Z', 'X', 'C', 'V', 'B', 'N', 'M']
                    ]
                ]
            ],
            switchKey: 0,
            spare: ['O', 'I'],
            special: ['临', '军', '警', '学', '使', '领'],
            routine: [],
            valueObj: new Array(8),
            valueIndex: 0
        }
    },
    watch: {
        visible: {
            handler (val) {
                if (val) {
                    document.body.style = 'overflow:hidden;'
                    let nv = new Array(8)
                    let index = 0
                    let lp = this.licensePlate.split('').filter(v => v)
                    lp.map((v, i) => {
                        nv[i] = v
                        index++
                    })
                    let nvlen = nv.filter(v => v).length
                    if (nvlen > 7) {
                        this.licensePlateType = 1
                        this.valueObj = nv
                    } else {
                        this.valueObj = nvlen ? nv.filter(v => v) : new Array(7)
                    }
                    this.valueIndex = index
                    if (this.valueIndex) {
                        this.onSwitchLang(1)
                    }
                } else {
                    document.body.removeAttribute('style')
                }
            },
            deep: true
        }
    },
    mounted () {
        this.$nextTick(() => {
            this.routine = this.keys[1][0]
        })
    },
    methods: {
        onChangeLicensePlateType () {
            this.licensePlateType = this.licensePlateType ? 0 : 1
            if (this.licensePlateType) {
                if (this.valueIndex <= 7) {
                    this.valueObj.push(null)
                }
            } else {
                if (this.valueIndex >= 8) {
                    this.valueObj.splice(7, this.valueObj.length - 7)
                    this.valueIndex = 7
                } else {
                    this.valueObj.splice(7, 1)
                }
            }
        },

        onSwitchLang (val) {
            this.$nextTick(() => {
                if (val) {
                    this.routine = this.keys[1][val]
                } else {
                    this.routine = this.keys[1][val]
                }
                this.switchKey = val
            })
        },

        onVisible (confirm = false) {
            if (confirm) {
                let value = this.valueObj.join('')
                if ((!this.licensePlateType && this.valueIndex < 7) || (this.licensePlateType && this.valueIndex < 8)) return
                this.$emit('confirm', value)
            } else {
                this.valueObj = new Array(8)
            }
            this.$emit('update:visible', false)
            this.valueIndex = 0
            this.onSwitchLang(0)
        },

        onInputLicensePlate (item, t = null) {
            this.$nextTick(() => {
                if (((!this.licensePlateType && this.valueIndex >= 7) || (this.licensePlateType && this.valueIndex >= 8)) && !t) return
                if (t) {
                    this.valueObj.push(item)
                }
                this.valueObj[this.valueIndex] = item;
                this.valueIndex += 1
                this.onSwitchLang(1)
            })
        },

        onRemoveLicensePlate () {
            if (!this.valueIndex) return
            this.valueObj[this.valueIndex - 1] = null
            let lpl = this.licensePlateType ? 8 : 7;
            if (this.valueIndex >= lpl) {
                this.valueObj.splice(this.valueIndex > lpl ? this.valueIndex - 1 : lpl, 1)
            }
            this.valueIndex -= 1
            if (!this.valueIndex) {
                this.onSwitchLang(0)
            }
        }
    }
}
</script>

<template>
    <transition name="collapse">
        <div v-if="visible" class="keyboard">
            <div class="keyboard-mask" @click="onVisible()"></div>
            <div class="keyboard-dialog">
                <div class="keyboard-content">
                    <div class="keyboard-header">
                        <div class="keyboard-header--inner" align="center">车牌号输入法</div>
                    </div>
                    <div class="keyboard-textarea">
                        <template v-for="(v, i) in valueObj">
                            <div v-if="i != valueObj.length">
                                <span v-if="v != null">{{ v }}</span>
                                <span v-else class="null">NULL</span>
                            </div>
                            <em v-if="i == 1"></em>
                        </template>
                    </div>
                    <div v-if="valueIndex >= 2" class="keyboard-number">
                        <div v-for="(item, index) in keys[0]" :key="index" class="key" @click="onInputLicensePlate(item)">
                            <span>{{ item }}</span>
                        </div>
                    </div>
                    <div class="keyboard-routine">
                        <div v-for="(line, lindex) in routine" :key="lindex" class="keyline">
                            <div v-for="(item, index) in line" :key="item" class="key" @click="onInputLicensePlate(item)">
                                <span>{{ item }}</span>
                            </div>
                            <template v-if="switchKey && valueIndex == 1 && !lindex">
                                <div v-for="(sitem, sidx) in spare" :key="sitem" class="key" @click="onInputLicensePlate(sitem)">
                                    <span>{{ sitem }}</span>
                                </div>
                            </template>
                        </div>
                        <div class="keyline" v-if="(!licensePlateType && valueIndex == 7) || (licensePlateType && valueIndex == 8)">
                            <div v-for="(item, index) in special" :key="item" class="key" @click="onInputLicensePlate(item, 's')">
                                <span>{{ item }}</span>
                            </div>
                        </div>
                        <div class="operas">
                            <div class="opera-key" @click="onChangeLicensePlateType" style="width:40%;">
                                <span>切换至{{ licensePlateType ? '普通车牌' : '新能源' }}</span>
                            </div>
                            <div class="opera-key delete" @click="onRemoveLicensePlate">
                                <span>删除</span>
                            </div>
                            <div class="opera-key">
                                <span @click="onVisible(true)">完成</span>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </transition>
</template>

<style scoped lang="less">
.collapse-enter-active,
.collapse-leave-active {
    opacity: 0;
    transition: opacity .15s;
}

.collapse-enter-to,
.collapse-leave {
    opacity: 1;
}

.keyboard {
    user-select: none;

    .keyboard-mask,
    .keyboard-dialog {
        position: fixed;
        z-index: 999;
    }

    .keyboard-mask {
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background-color: rgba(0, 0, 0, .5);
    }

    .keyboard-dialog {
        bottom: 0;
        left: 0;
        width: 100%;
        user-select: none;
    }

    .keyboard-header--inner {
        padding: 10px 0 30px;
        color: #777;
        font-size: 30px;
    }

    .keyboard-content {
        background-color: #f5f5f5;
        padding: 20px;
        text-align: center;
        box-shadow: 0 4px 10px 0 rgba(0, 0, 0, .5);

        .keyboard-textarea,
        .keyboard-number,
        .operas,
        .keyline {
            display: flex;
            align-items: center;
        }

        .keyboard-textarea {
            background-color: white;
            margin: 0 -20px 30px;
            padding: 20px;

            div {
                flex: 1;
                padding: 0 10px;

                span {
                    line-height: 70px;
                    // height: 70px;
                    font-size: 36px;
                    // background-color: white;
                    display: block;
                    border-bottom: 2px solid green;
                    // box-shadow: 0 4px 10px 0 rgba(0, 0, 0, .25);
                    // border-radius: 10px;
                }

                .null {
                    font-size: 12px;
                    color: #ffffff;
                }
            }

            em {
                display: inline-block;
                width: 10px;
                height: 10px;
                background-color: rgb(194, 194, 194);
                border-radius: 50%;
                margin: 0 10px
            }
        }

        .opera-key,
        .key {
            font-size: 34px;

            &:active {
                background-color: #f8f8f8;
            }

            span {
                line-height: 1.8;
            }
        }

        .keyboard-number {
            flex-wrap: nowrap;
            justify-content: center;

            .key {
                width: calc(100% / 10);
                padding: 0 6px;
                margin-bottom: 14px;

                span {
                    display: block;
                    background-color: white;
                    border-radius: 10px;
                    box-shadow: 0 2px 6px 0 rgba(0, 0, 0, .25);
                }
            }
        }

        .keyboard-routine {
            .operas,
            .keyline {
                justify-content: center;

                .opera-key,
                .key {
                    width: 10%;
                    padding: 0 6px;
                    margin-bottom: 14px;

                    span {
                        display: block;
                        background-color: white;
                        border-radius: 10px;
                        box-shadow: 0 4px 8px 0 rgba(0, 0, 0, .25);

                        &:active {
                            background-color: rgb(230, 230, 230);
                        }
                    }
                }

                .opera-key {
                    width: 20%;

                    span {
                        font-size: 30px;
                        line-height: 2.2;
                    }
                }

                .opera-key.switch-key {
                    .en {
                        em {
                            &:first-child {
                                font-size: 80%;
                                color: #777;
                            }
                        }
                    }
                    .cn {
                        em {
                            &:last-child {
                                font-size: 80%;
                                color: #777;
                            }
                        }
                    }
                }

                .opera-key.delete {
                    color: red;
                }
            }

            .operas {
                border-top: 1px solid #dddee1;
                margin-top: 10px;
                padding-top: 20px;
            }
        }
    }
}
</style>