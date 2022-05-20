<script>
/**
 * 仿 weui datepicker scroll 事件
 */

const formatNumber = n => {
    return n.toString().length > 1 ? n : '0' + n
}

const getCurrentDatetime = type => {
    const D = new Date()
    const year = D.getFullYear()
    const month = D.getMonth() + 1
    const date = D.getDate()
    const hours = D.getHours()
    const minute = D.getMinutes()
    return !type ? [
        [year, formatNumber(month), date].join('/'),
        [formatNumber(hours), formatNumber(minute)].join(':')
    ].join(' ') : [year, formatNumber(month), date, formatNumber(hours), formatNumber(minute)]
}


const setTransition = ($target, time) => {
    $target.style.transition = `all ${time}s`;
    $target.style.webkitTransition = `all ${time}s`;
}

const setTranslate = ($target, t) => {
    $target.style.transform = `translate3d(0, ${t}px, 0)`;
    $target.style.webkitTransform = `translate3d(0, ${t}px, 0)`;
}

const getRemHeight = () => {
    let htmlFontSize = getComputedStyle(document.documentElement)['font-size']
    return +htmlFontSize.slice(0, htmlFontSize.indexOf('px'))
}

const scroll = options => {
    const { el, current } = options
    let rowHeight = getRemHeight();
    let offset = 2;
    let start;
    let end;
    let startTime;
    let endTime;
    let translate;
    let lastIndex = current;
    let translateTarget = '.nx-column--content';
    const points = [];

    if (current) {
        translate = -(current - offset) * rowHeight;
    } else {
        translate = offset * rowHeight;
    }
    setTranslate(el.querySelector(translateTarget), translate);


    const _stop = translateY => {
        translate += translateY;

        translate = Math.round(translate / rowHeight) * rowHeight;
        const max = offset * rowHeight
        const min = -(rowHeight * (options.items.length - offset - 1))
        if (translate > max) {
            translate = max;
        }
        if (translate < min) {
            translate = min;
        }

        let index = offset - translate / rowHeight;
        if (options.disabled.value) {
            index = parseInt(index);
            while (!!options.items[index] && options.items[index] < options.disabled.r[options.disabled.column]) {
                translateY > 0 ? ++index : --index;
            }
        }
        translate = (offset - index) * rowHeight;
        setTransition(el.querySelector(translateTarget), .3);
        setTranslate(el.querySelector(translateTarget), translate);

        if (index !== lastIndex) {
            options.onChange.call(this, options.items[index], index)
        }
        lastIndex = null
    }

    const _start = pageY => {
        start = pageY
        startTime = +new Date()
    }

    const _move = pageY => {
        end = pageY;
        let newTranslate = translate + (end - start);
        setTransition(el.querySelector(translateTarget), 0)
        setTranslate(el.querySelector(translateTarget), newTranslate)

        startTime = +new Date()
        points.push({ time: startTime, y: end });

        if (points.length > 40) {
            points.shift()
        }

        newTranslate = Math.round(newTranslate / rowHeight) * rowHeight
        const max = offset * rowHeight
        const min = -(rowHeight * (options.items.length - offset - 1))
        if (newTranslate > max || newTranslate < min) return;

        const index = offset - newTranslate / rowHeight
        if (index != lastIndex) {
            options.onChange.call(this, options.items[index], index)
        }
    }

    const _end = pageY => {
        if (!start) return;
        const endTime = new Date().getTime();
        end = pageY

        const relativeY = el.getBoundingClientRect().top + 5 * rowHeight / 2;

        if (endTime - startTime > 100) {
            if (Math.abs(end - start) > 10) {
                _stop(end - start)
            } else {
                _stop(relativeY - end)
            }
        } else {
            if (Math.abs(end - start) > 10) {
                const endPos = points.length - 1;
                let startPos = endPos
                for (let i = endPos; i > 0 && startTime - points[i].time < 100; i--) {
                    startPos = i;
                }

                if (startPos != endPos) {
                    const ep = points[endPos];
                    const sp = points[startPos];
                    const t = ep.time - sp.time;
                    const s = ep.y - sp.y;
                    const v = s / t;
                    _stop(v * 150 + (end - start));
                } else {
                    _stop(0)
                }
            } else {
                _stop(relativeY - end)
            }
        }
        start = null;
    }

    el.addEventListener('touchstart', e => _start(e.changedTouches[0].pageY))
    el.addEventListener('touchmove', e => _move(e.changedTouches[0].pageY))
    el.addEventListener('touchend', e => _end(e.changedTouches[0].pageY))
}

export default {
    props: {
        value: {
            type: String,
            default: null
        },
        disabledBefore: {
            type: Boolean,
            default: false
        }
    },
    data: () => ({
        datetime: '',
        visible: false,
        columns: [[], [], [], [], []],
        values: [0, 0, 0, 0, 0],
        current: getCurrentDatetime('array')
    }),
    watch: {
        visible: {
            handler (visit) {
                if (visit) {
                    document.body.style.overflow = 'hidden';
                    if (!this.value) {
                        this.init()
                    }
                    this.$nextTick(() => this.addTouch())
                } else {
                    document.body.removeAttribute('style');
                    // this.$nextTick(() => this.removeTouch())
                }
            },
            deep: true
        }
    },
    mounted () {
        this.$nextTick(() => {
            this.datetime = this.value || getCurrentDatetime()
            this.columns[0] = this.getDateNumber(2000, 2030)
            this.columns[1] = this.getDateNumber(1, 12)
            this.columns[2] = this.calculLeapDay()
            this.columns[3] = this.getDateNumber(0, 23)
            this.columns[4] = this.getDateNumber(0, 59)
        })
    },

    methods: {
        init () {
            getCurrentDatetime('array').map((item, index) => {
                this.values[index] = this.columns[index].indexOf(item)
            })
        },

        addTouch () {
            let startTime, endTime;
            let page = {}
            let that = this;
            this.columns.map((col, index) => {
                scroll({
                    el: this.$refs[`columnRef${index}`][0],
                    items: col,
                    current: this.values[index],
                    disabled: {
                        value: this.disabledBefore,
                        column: index,
                        r: this.current
                    },
                    onChange (value, idx) {
                        that.values[index] = idx
                    }
                });
            })
        },

        getDateNumber (start, end) {
            let nums = []
            for (let i = start; i <= end; i++) {
                nums.push(formatNumber(i))
            }
            return nums
        },
        
        calculLeapDay (year = null, month = null) {
            let date = 28
            if (month != 2) {
                if ([1, 3, 5, 7, 8, 10, 12].indexOf(month) != -1) {
                    date = 31
                } else {
                    date = 30
                }
            } else {
                if ((year % 4 == 0 && !(year % 100 == 0)) || year % 400 == 0) {
                    date = 29
                }
            }
            let dates = []
            for (let i = 1; i <= date; i++) {
                dates.push(formatNumber(i))
            }
            return dates
        },

        onConfirm () {
            let indexItem = []
            this.values.map((item, index) => {
                indexItem.push(this.columns[index][item])
            })
            this.datetime = [
                [indexItem[0], indexItem[1], indexItem[2]].join('/'),
                [indexItem[3], indexItem[4]].join(':')
            ].join(' ')
            this.visible = false;
            this.$emit('input', this.datetime)
        }
        
    }
}
</script>

<template>
    <div class="picker">
        <div class="picker-inner" @click="visible = true">
            <span>{{ datetime }}</span>
            <span class="arrow-right"></span>
        </div>

        <transition name="picker">
            <div v-if="visible" class="nx-dialog">
                <div class="nx-dialog--mask" @click="visible = false"></div>
                <div class="nx-dialog--body">
                    <div class="nx-dialog--content">
                            <div class="nx-dialog--hd">
                            <div class="nx-dialog--hd-inner">时间选择器</div>
                        </div>
                        <div class="nx-dialog--bd">
                            <div v-for="(col, ci) in columns" :key="ci" class="nx-picker--column" :ref="'columnRef' + ci">
                                <div class="nx-column--mask"></div>
                                <div class="nx-column--content">
                                    <div v-for="(item, index) in col" :key="index" :class="{
                                        'disabled': disabledBefore && item < current[ci]
                                    }" class="nx-column--item">{{ item }}</div>
                                </div>
                            </div>
                        </div>
                        <div class="nx-dialog--ft">
                            <a href="javascript:;" class="nx-dialog--btn nx-dialog--cancel" @click="visible = false">取消</a>
                            <a href="javascript:;" class="nx-dialog--btn nx-dialog--confirm" @click="onConfirm">确定</a>
                        </div>
                    </div>
                </div>
            </div>
        </transition>
    </div>
</template>

<style lang="less" scoped>
.picker {
    .nx-dialog {
        user-select: none;

        .nx-dialog--mask,
        .nx-dialog--body {
            position: fixed;
            z-index: 999;
            width: 100%;
        }

        .nx-dialog--mask {
            top: 0;
            left: 0;
            height: 100%;
            background-color: rgba(0, 0, 0, .75);
        }

        .nx-dialog--body {
            bottom: 0;
            left: 0;

            .nx-dialog--content {
                background-color: white;
                padding: .25rem;
                border-radius: .4rem .4rem 0 0;

                .nx-dialog--hd {
                    text-align: center;
                    padding: .25rem 0;
                }

                .nx-dialog--bd {
                    height: 5rem;
                    display: flex;
                    justify-content: center;
                    margin: .5rem 0;
                    overflow: hidden;

                    .nx-picker--column {
                        flex: 1;
                        text-align: center;
                        height: 100%;
                        position: relative;
                        -webkit-overflow-scrolling: touch;
                        // overflow-y: auto;
                        // overflow-x: hidden;

                        .nx-column--mask {
                            position: absolute;
                            top: 0;
                            left: 0;
                            width: 100%;
                            height: 100%;
                            margin: 0 auto;
                            z-index: 3;
                            background-image: -webkit-linear-gradient(top,rgba(255,255,255,0.95),rgba(255,255,255,0.6)),-webkit-linear-gradient(bottom,rgba(255,255,255,0.95),rgba(255,255,255,0.6));
                            background-image: linear-gradient(180deg,rgba(255,255,255,0.95),rgba(255,255,255,0.6)),linear-gradient(0deg,rgba(255,255,255,0.95),rgba(255,255,255,0.6));
                            background-position: top,bottom;
                            background-size: 100% 2rem;
                            background-repeat: no-repeat;
                            -webkit-transform: translateZ(0);
                            transform: translateZ(0);
                        }

                        .nx-column--content {
                            position: absolute;
                            width: 100%;

                            .nx-column--item {
                                line-height: 1rem;
                            }

                            .nx-column--item.disabled {
                                color: #999;
                            }
                        }
                    }
                }

                .nx-dialog--ft {
                    text-align: center;

                    .nx-dialog--btn {
                        display: inline-block;
                        width: 4rem;
                        background-color: #e6e6e6;
                        height: 1rem;
                        line-height: 1rem;
                        color: #676767;
                        border-radius: .15rem;
                        margin: 0 .25rem;
                        font-size: .4rem;
                    }

                    .nx-dialog--confirm {
                        background-color: var(--theme-color);
                        color: white;
                    }
                }
            }
        }
    }

    .picker-inner {
        height: 1rem;
        line-height: 1rem;
        position: relative;
        padding-right: .5rem;

        .arrow-right {
            width: .2rem;
            height: .2rem;
            border-style: solid;
            border-width: 2px;
            border-color: transparent black black transparent;
            position: absolute;
            top: 50%;
            right: 0;
            transform: translateY(-50%) rotate(-45deg);
        }
    }
}

// .picker-enter-active,
// .picker-leave-active {
//     transition: all .2s;
// }

// .picker-enter,
// .picker-leave-to {
//     opacity: 0;
//     z-index: 999;
// }
</style>