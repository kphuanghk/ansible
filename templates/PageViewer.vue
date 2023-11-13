<template>
  <teleport to="body">
    <div
      class="absolute left-0 top-0 z-20 w-1/2 h-screen overflow-hidden bg-white"
    >
      <div
        v-if="loading"
        class="text-4xl font-bold animate-bounce w-full h-full flex justify-center items-center"
      >
        Loading..
      </div>
      <iframe
        v-show="!loading"
        src="blank.html"
        type="application/pdf"
        ref="pdfviewframe"
        width="100%"
        height="100%"
        class="bg-red-200 object-center"
      ></iframe>
      <div
        class="absolute w-full bottom-0 left-0 h-10 flex flex-row bg-blue-200 text-gray-600 font-semibold items-center justify-between"
      >
        <div
          @click="prevPage"
          class="cursor-pointer hover:bg-blue-500 hover:text-white mx-3 px-3 py-1 rounded-md"
        >
          ◀️ Previous
        </div>
        <div>{{ currentPage + 1 }} / {{ file.pages.length }}</div>
        <div
          @click="nextPage"
          class="cursor-pointer hover:bg-blue-500 hover:text-white mx-3 px-3 py-1 rounded-md"
        >
          Next ▶️
        </div>
        <button @click="$emit('close')" class="mr-3 px-2 bg-blue-400 p-1">
          Close
        </button>
      </div>
    </div>
    <div
      class="absolute right-0 top-0 z-20 w-1/2 h-screen overflow-hidden bg-white"
      ref="canvasparent"
    >
      <canvas width="1500" height="1800" ref="pagecanvas"></canvas>

      <div
        class="absolute w-full bottom-0 right-0 h-10 flex flex-row bg-blue-200 text-gray-600 font-semibold items-center justify-between"
      >
        <div>
          Font Size : <input class="w-12 p-2 text-left" v-model="fontsize" />
        </div>
        <button
          @click="fLoadPdf(currentPage)"
          class="mr-3 px-2 bg-blue-400 p-1"
        >
          Apply
        </button>
      </div>
    </div>
  </teleport>
</template>

<script setup lang="ts">
import {
  Canvas,
  Point,
  TPointerEvent,
  TPointerEventInfo,
  Textbox
} from 'fabric'
import { blob, json } from '../utils'
import { ref, onMounted, PropType } from 'vue'

const loading = ref(true)
const pdfviewframe = ref<HTMLIFrameElement>()
const canvasparent = ref<HTMLDivElement>()
const pagecanvas = ref<HTMLCanvasElement>()
const fontsize = ref(12)
let canvas: Canvas | null = null

const props = defineProps({
  file: {
    type: Object as PropType<{ pages: string[]; texts: string[] }>,
    required: true
  }
})

defineEmits(['close'])

const currentPage = ref(0)

const nextPage = async () => {
  if (currentPage.value + 1 == props.file.pages.length) {
    return
  }
  currentPage.value = currentPage.value + 1
  await fLoadPdf(currentPage.value)
}

const prevPage = async () => {
  if (currentPage.value == 0) {
    return
  }
  currentPage.value = currentPage.value - 1
  await fLoadPdf(currentPage.value)
}

const fLoadPdf = async (idx: number = 0) => {
  const pdf = props.file.pages[idx]
  const jsonfile = pdf.replace('pdf', 'json')
  const { data: binary } = await blob.post('/sample/getpdf', { dataFile: pdf })
  const content = URL.createObjectURL(binary)
  pdfviewframe.value!.src = content
  setTimeout(() => (loading.value = false), 500)
  const { data: jsontext } = await json.post('/sample/gettext', {
    dataFile: jsonfile
  })
  const generaltext = jsontext as GeneralText
  const { height, width } = generaltext.result
  console.log({ height, width })

  const lines = generaltext.result.lines
  console.log('Start to fill the text')
  canvas?.clear()
  lines.forEach((line, _idx) => {
    const tb = new Textbox(line.text, {
      left: line.position[0],
      top: line.position[1],
      width: line.position[2] - line.position[0],
      fontSize: fontsize.value,
      cornerColor: 'blue',
      color: 'blue'
    })

    tb.onSelect = (option: { e: PointerEvent }) => {
      console.log('Selected the text..' + option, line.text)
      navigator.clipboard.writeText(line.text)
      return false
    }
    //console.log('Add text ', idx, line.text)
    canvas?.add(tb)
  })
}

onMounted(async () => {
  canvas = new Canvas(pagecanvas.value!)

  canvas.on('mouse:wheel', function (opt) {
    const delta = opt.e.deltaY
    let zoom = canvas!.getZoom()
    zoom *= 0.999 ** delta
    if (zoom > 20) zoom = 20
    if (zoom < 0.01) zoom = 0.01
    const zoomPoint: Point = new Point(opt.e.offsetX, opt.e.offsetY)
    canvas!.zoomToPoint(zoomPoint, zoom)
    opt.e.preventDefault()
    opt.e.stopPropagation()
    var vpt = canvas!.viewportTransform
    if (zoom < 400 / 1000) {
      vpt[4] = 200 - (1000 * zoom) / 2
      vpt[5] = 200 - (1000 * zoom) / 2
    } else {
      if (vpt[4] >= 0) {
        vpt[4] = 0
      } else if (vpt[4] < canvas!.getWidth() - 1000 * zoom) {
        vpt[4] = canvas!.getWidth() - 1000 * zoom
      }
      if (vpt[5] >= 0) {
        vpt[5] = 0
      } else if (vpt[5] < canvas!.getHeight() - 1000 * zoom) {
        vpt[5] = canvas!.getHeight() - 1000 * zoom
      }
    }
  })

  canvas.on('mouse:down', function (opt) {
    var evt = opt.e
    if (evt.altKey === true) {
      // @ts-ignore
      this.isDragging = true
      // @ts-ignore
      this.selection = false
      // @ts-ignore
      this.lastPosX = evt.clientX
      // @ts-ignore
      this.lastPosY = evt.clientY
    }
  })
  canvas.on('mouse:move', function (opt) {
    // @ts-ignore
    if (this.isDragging) {
      var e = opt.e
      // @ts-ignore
      var vpt = this.viewportTransform
      // @ts-ignore
      vpt[4] += e.clientX - this.lastPosX
      // @ts-ignore
      vpt[5] += e.clientY - this.lastPosY
      // @ts-ignore
      this.requestRenderAll()
      // @ts-ignore
      this.lastPosX = e.clientX
      // @ts-ignore
      this.lastPosY = e.clientY
    }
  })

  canvas.on('mouse:up', function (_) {
    // on mouse up we want to recalculate new interaction
    // for all objects, so we call setViewportTransform
    // @ts-ignore
    this.setViewportTransform(this.viewportTransform)
    // @ts-ignore
    this.isDragging = false
    // @ts-ignore
    this.selection = true
  })

  await fLoadPdf()
})
</script>
