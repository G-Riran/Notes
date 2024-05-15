<script>
    const isDisplay = ref(true)
    const Switch = () => {
      isDisplay.value = !isDisplay.value
    }
</script>
<template>
    <p v-show: isDisplay>Hello</p>
    <button @click: Switch>switch</button>
</template>

PS:
1、v-show v-if通用

2、const let通用  
    const：可改变其值，但无法改变它的引用  
    let：可重新赋值，但vue会无法追踪，导致视图无法更新  
    改用例中建议使用const  

3、ref reactive通用
    ref：创建单个可相应的基本值
    reactive：创建一个包含多个属性的可相应对象
    该用例中建议使用ref

    使用reactive需要注意的地方：
        
        const isDisplay = reactive({
          Display: true
        })

        const Switch = () => {
            isDisplay.Display = !isDisplay.Display
        }

        <p v-show=isDisplay.Display>Hello</p>