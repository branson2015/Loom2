<template>
  <div 
    id="loom"
    :style="{
      height: IS_config ? IS_config.info.window.height + 'px' : '0px',
      width:  IS_config ? IS_config.info.window.width  + 'px' : '0px',
      position: 'absolute',
      'margin-left': 'auto',
      'margin-right': 'auto',
      'margin-top': '25px',
      left: 0,
      right: 0,
      'text-align': 'center',
      'pointer-events': 'none'
    }"
  >
    <!-- global list of video tags!-->
    <video 
      v-for="(val, key) in videoPlayers"
      :key="key"
      :ref="el => { if(el) videoPlayerRefs[val.id] = el }"
      :style="{
        width:  IS_config ? IS_config.info.window.width  + 'px' : '0px',
        height: IS_config ? IS_config.info.window.height + 'px' : '0px',
      }"
      class="video-js" 
      crossorigin="anonymous"
      style="position: absolute; top: 0px; left: 0px; visibility: hidden;"
    />
    <loom-video-canvas
      v-for="videoTarget in currentVideoTargets"
      :key="`${renderAppMode}-${videoTarget.page}-${videoTarget.id}`"
      :ref="el => { if(el) loomVideoCanvasRefs[videoTarget.id] = el }"
      :instanceID="id"
      :targets="videoTarget.targets || targets"
      :targetData="videoTarget"
      :start_state_id="current_state.id"
      :moveRegionMode="moveRegionMode"
      :frameLinkMode="frameLinkMode"
      :overlay="overlay"
      :renderMode="renderMode"
      :renderAppMode="renderAppMode"
      :currentConfig="currentConfig"
      :info="IS_config.info"
      :editRegionMode="editRegionMode"
      @frame_processed="() => videoTarget.processed = true"
    />
  </div>
</template>


<script>
import videojs from 'video.js'
import 'video.js/dist/video-js.min.css'

import loomConfig from './loomConfig.json'
import transformMode from './transforms.js'
import loomVideoCanvas from './loomVideoCanvas.vue'
import utils from './utils/utils.js'

import { ref, reactive, onMounted, computed, provide, inject, nextTick } from 'vue'

export default {
  name: 'LoomInstance',
  components: { loomVideoCanvas },
  props: {
    renderMode: String,
    videoTargetCache: {
      type: Object,
      default: {}
    },
    video_filename: {
      type: String,
      default: 'video.mp4'
    },
    directory: String,
    moveRegionMode: Boolean,
    frameLinkMode: Boolean,
    editRegionMode: Boolean,
    overlay: Boolean,
    id: String,
    name: String,
  },
  setup(props, context){
    //global
    const emitter = inject("emitter");
    const {appConfig, linkedCanvases, appRefs, zip} = inject('manager');


    //initialization
    const loaded = ref(false);
    const renderAppMode = ref(null);
    const IS_config = ref(null);
    const currentConfig = ref(null);
    const createNewVideoPlayers = (src) => {
      for(let i = 0; i < numVideoPlayers.value; ++i)  createNewVideoPlayer(i, src);
    }
    const createNewVideoPlayer = (id, src) => {
      const vp = {id, promises: []};
      videoPlayers.push(vp);
      nextTick(()=>{
        const player = setupVideo(videoPlayerRefs[id], src);
        if(player) vp.player = player;
        else  console.error("No Player!");
      });
    };
    const setupVideo = (element, src) => {
      const p = videojs.getPlayer(element);
      if(p) return p;

      const videoOptions = {
        sources: [{
          src: "data:video/mp4;base64,"+btoa(src),
          type: 'video/mp4',
        }],
        controls: false,
        loadingSpinner: false,
        controlBar: false,
      };
      return videojs(element, videoOptions, function() {  
        this.on("click", function(ev) {
            ev.preventDefault();
        });
      });
    };
    const load = () => {
      if(loaded.value) return Promise.resolve([IS_config, appConfig.value]);
      return fetch(props.directory + '.rva').then(res => res.blob())
      .then(async blob => {
        const zipReader = new zip.ZipReader(new zip.BlobReader(blob));

        const entries = await zipReader.getEntries();
        const src = await entries[1].getData(new zip.BlobWriter());
        
        let reader = new FileReader();
        reader.onloadend = (e) => createNewVideoPlayers(e.target.result);
        reader.readAsBinaryString(src);

        const IS_config = entries[0].getData(new zip.TextWriter());
        return IS_config;
      })
      .then(IS_config => JSON.parse(IS_config)).catch(e => null)
      .then(nIS_config => {
        if(nIS_config == null){
          console.error('ERROR, IS_config not loaded\n');
          return [null, null];
        }
        if(nIS_config.info['version'] != loomConfig['version']){
          console.error('ERROR, out of version IS_config file, run Recorder/convert.py');
          return [null, null];
        }
        IS_config.value = nIS_config;
        currentConfig.value = IS_config.value.hasOwnProperty(renderAppMode.value) ? IS_config.value[renderAppMode.value] : IS_config.value['original'];
        loaded.value = true;
        transformedTargetCache['hidden'] = {};
        props.videoTargetCache['hidden'] = {};
        transformedTargetCache['original'] = IS_config.value['original'].data;
        return [IS_config, appConfig.value];
      });
    };
    const init = (m) => {
      renderAppMode.value = `${m.value}_${m.renderMode}`;
      return load().then(([IS_config, ISL])=>{
        if(IS_config === null || ISL === null) return;
        if(!m.selected) renderAppMode.value = 'hidden';

        targetCacheModeChange(renderAppMode.value, m.renderMode);
        init_videoTargetCache(ISL, m.value, m.renderMode);
        videoCacheModeChange(renderAppMode.value);

        current_state.value = Object.values(targets.value).filter(t => t.frame_no > 0).sort((a,b) => a.frame_no - b.frame_no)[0];
        changeState(current_state.value.id);

        //change frame to be mid-framelinked
        const ld = Object.values(linkedCanvases.value.sets).find(l => l.data.instance == props.id);
        if(ld){
          const l = ld.data;
          emitter.emit('changeVideoFrame', [l.instance, l.page, l.vcid, -1, l.frame, false]);
        }
      });
    };
    const init_videoTargetCache = (cvt_config, renderAppMode, renderMode) => {
      const rarm = `${renderAppMode}_${renderMode}`
      if(!cvt_config || cvt_config[rarm] === undefined) return;
      const vt_config = cvt_config[rarm]
      const val = utils.deepCopy(vt_config);
      for(const page of Object.values(val)){
        for(const vc of Object.values(page)){
          vc.processed = vc.processed === undefined ? Boolean(vc.cutouts && vc.cutouts.length > 0) : vc.processed;
          vc.cutouts = vc.cutouts || [];
          const instance = renderAppMode
          const lfs = vc.linkedFrames || [];
          for(const lf of lfs){
            /* if(lf.mode != renderMode)
              continue; */
            const ld = {
              mode: renderMode,
              instance: instance,
              page: vc.page,
              vcid: vc.id,
            };
            /*if(lf.instance == ld.instance){
              const linkFromName = `${ld.instance}_${ld.page}_${ld.vcid}_all`;
              const linkToName   = `${lf.instance}_${lf.page}_${lf.vcid}_all`;
              if(!linkedCanvases.value.exists(linkFromName))  linkedCanvases.value.add(linkFromName, ld);
              if(!linkedCanvases.value.exists(linkToName  ))  linkedCanvases.value.add(linkToName  , lf);
              linkedCanvases.value.merge(linkFromName, linkToName);
            } else {*/
              for(const frame of lf.frames){
                const ldc = {...ld, frame: frame[0]};
                const lfc = {...lf, frame: frame[1]};
                const linkFromName = `${ldc.instance}_${ldc.page}_${ldc.vcid}_${ldc.frame}`;
                const linkToName   = `${lfc.instance}_${lfc.page}_${lfc.vcid}_${lfc.frame}`;
                if(!linkedCanvases.value.exists(linkFromName))  linkedCanvases.value.add(linkFromName, ldc);
                if(!linkedCanvases.value.exists(linkToName  ))  linkedCanvases.value.add(linkToName  , lfc);
                linkedCanvases.value.merge(linkFromName, linkToName);
              }
            // }
          }
          if(vc.parentCanvas && vc.parentCanvas != '-1'){
            const pc = loomVideoCanvasRefs[vc.parentCanvas];
            if(vc.makeCutout && pc){
              pc.targetData.cutouts = vc.parentCanvas.cutouts || [];
              const region = vc.region;
              const minX = Math.min(...region.map(p => p.x));
              const maxX = Math.max(...region.map(p => p.x));
              const minY = Math.min(...region.map(p => p.y));
              const maxY = Math.max(...region.map(p => p.y));
              const pcutouts = pc.targetData.cutouts;
              pcutouts.push({
                poly: region,
                width: maxX-minX,
                height: maxY-minY,
                top: minY,
                left: minX,
                id: vc.id
              });
            }
          }
        }
      }
      for(let key in vt_config){
        props.videoTargetCache[rarm][key] = vt_config[key];
      }
    };
    const videoCacheModeChange = (mode) => { if(!props.videoTargetCache.hasOwnProperty(mode))  newVideoTarget({processed: false}, mode); };


    //state changing
    const current_state = ref(null);
    const interactionHistory = reactive([]);
    const maxHistoryLength = ref(30);
    const changeState = (target_id, offset = 0) => {
      const target = targets.value[target_id];
      if(target === undefined)  { console.error("Invalid state");   return; }
      changeStateWithFrameNo(target.frame_no, offset);
    };
    const changeStateWithFrameNo = (frame) => { current_state.value = Object.values(targets.value).find(o => o.frame_no == frame); };
    const targetCacheModeChange = (mode, transform) => {
      if(!transformedTargetCache.hasOwnProperty(mode)){
        if(!IS_config.value.hasOwnProperty(mode)){
          IS_config.value[mode] = transformMode(IS_config.value, transformedTargetCache['original'], transform);
        } 
        transformedTargetCache[mode] = IS_config.value[mode].data || {};
      }
    };
    const updateInteractionHistory = (target, e, videoCanvas) => {
      const interaction = { id: target.id, videoCanvas, target, event: e };
      interactionHistory.push(interaction);
      if(interactionHistory.length > maxHistoryLength.value)  interactionHistory.shift();
      emitter.emit("runInteractionAnalysis", interactionHistory);
    };


    //frame changing
    const fps = ref(30);
    const numVideoPlayers = ref(3);
    const videoPlayers = reactive([]);
    const lastUsedVideoPlayer = ref(0);
    const videoPlayerRefs = reactive({});
    const changeVideoFrame = (page, videoCanvasID, frameNo, emit=true) => {
      if(!frameNo && frameNo !== 0) {
        loomVideoCanvasRefs[videoCanvasID].draw(null, emit);
        return;
      }
      const vp = videoPlayers[lastUsedVideoPlayer.value];
      if(!vp) return;
      lastUsedVideoPlayer.value = (lastUsedVideoPlayer.value+1)%numVideoPlayers.value;
      return new Promise((res, rej) => {
        vp.promises.push(res);
        if(!vp.inUse){
          res(res);
        }
        setTimeout(() => rej(res), 1000);
      }).then(resolve => {
        const pi = vp.promises.findIndex(res => res == resolve);
        vp.player.on('timeupdate', function timeUpdate(){
          const vpEl = videoPlayerRefs[vp.id];
          if(!vpEl) return;
          if(loomVideoCanvasRefs[videoCanvasID])  loomVideoCanvasRefs[videoCanvasID].draw(vpEl, emit);
          vp.player.off('timeupdate', timeUpdate);
          if(vp.promises[pi+1]) vp.promises[pi+1](vp.promises[pi+1]);
          vp.inUse = false;
          vp.promises.splice(pi, 1);
        });
        vp.inUse = true;
        vp.player.currentTime(frameNo/fps.value);
      }).catch(resolve => {
        const pi = vp.promises.findIndex(res => res == resolve);
        vp.promises.splice(pi, 1);
        console.warn("frame took too long");
        //todo: retry?
      });
    };


    //frame linking
    const addInstanceLink = (fl, ld) => {
      const fli = appRefs[fl.instance].renderAppMode;
      if(props.videoTargetCache[fli][fl.page][fl.vcid].linkedFrames == undefined)
        props.videoTargetCache[fli][fl.page][fl.vcid].linkedFrames = [];
      const linkedFrames = props.videoTargetCache[fli][fl.page][fl.vcid].linkedFrames;
      const lf = linkedFrames.find(lf => lf.mode == ld.mode && lf.page == ld.page && lf.vcid == ld.vcid);
      if(lf != undefined){
        lf.frames = lf.frames || [];
        lf.frames.push([fl.frame, ld.frame]);
      } else {
        const cld = {...ld}; //shallow copy
        if(fl.instance != ld.instance){
          delete cld.frame;
          cld.frames = [[fl.frame, ld.frame]];
        }
        linkedFrames.push(cld)
      }
    };


    //target components
    const targets = computed(() => {
      const originalTargets = utils.shallowCopy(transformedTargetCache['original']);
      delete originalTargets.mode;
      const newTargets = utils.shallowCopy(transformedTargetCache[renderAppMode.value] || {});
      delete newTargets.mode;
      const mergedTargets = utils.deepMerge(originalTargets, newTargets);
      return mergedTargets;
    });


    //video targets
    const loomVideoCanvasRefs = reactive({});
    const transformedTargetCache = reactive({});
    const computeCurrentVideoTargets = computed(() => {
      if(!current_state.value || !props.videoTargetCache[renderAppMode.value]) return [null, null];
      const vts = Object.entries(props.videoTargetCache[renderAppMode.value]);
      let vt = null;
      let page = '-1';
      //traverse up parents
      for(let cs = current_state.value; vt === null && cs && cs.id != cs.parent_id; cs = targets.value[cs.parent_id]){
        if(typeof cs.frame_no === "string" && cs.frame_no.includes("frameless")) continue;
        //check if any of our videoTargets is an ancestor of our current state
        for(let i = 0; i < vts.length; ++i){
          if(String(cs.id) === vts[i][0]){
            vt = vts[i][1];
            page = vts[i][0];
            break;
          }
        }
      }
      if(vt === null){
        vt = props.videoTargetCache[renderAppMode.value]['-1'];   //return root
      }
      return [page, vt];
    });
    const currentVideoTargets = computed(() => { return computeCurrentVideoTargets.value[1]; });
    const currentPage = computed(() => { return computeCurrentVideoTargets.value[0]; });
    const newVideoTarget = async (obj={}, mode=undefined, clear=false) => {
      if(mode === undefined)  mode = renderAppMode.value;
      let page = '-1';
      let csid = 0;
      if(!props.videoTargetCache.hasOwnProperty(mode)){
        props.videoTargetCache[mode] = {};
      } else { 
        if(obj.newPageFromRoot === false){
          page = currentPage.value;
        }else{
          page = String(current_state.value.id); 
        }
        csid = page;
      }
      const region = utils.rectToPoly({
        x: 0, y: 0,
        width: IS_config.value.info.window.width, 
        height: IS_config.value.info.window.height,
      });
      if(!props.videoTargetCache[mode][page] || clear)  props.videoTargetCache[mode][page] = {};
      const id = String(Object.keys(props.videoTargetCache[mode][page]).length);

      props.videoTargetCache[mode][page][id] = {
        page,
        id,
        current_state_id: csid,
        region,
        top: 0,
        left: 0,
        makeCutout: false,
        parentCanvas: false,
        cutouts: [],
        startupFn: null,
        processed: true,
        ...obj //to overwrite preceding values
      };
      return await nextTick(() => {
        return loomVideoCanvasRefs[id];
      });
    };
    const destroyVideoTarget = async (mode, page, id) => {
      delete props.videoTargetCache[mode][page][id];
    }


    //operations
    const Up = (videoCanvas) => {
      videoCanvas.zIndex++;
    }
    const Down = (videoCanvas) => {
      videoCanvas.zIndex--;
    }
    const Delete = (videoCanvas) => {
      let id = null;
      if(videoCanvas === null && currentVideoTargets.value !== null)  id = '-1';
      else if(videoCanvas)  id = videoCanvas.id;
      if(id !== null)  delete currentVideoTargets.value[id];
    };
    const paste = (pasteBin) => {
      if(!pasteBin) return;
      const fn = pasteBin.startupFn;
      const copy = utils.deepCopy(pasteBin);
      copy.startupFn = (c) => { c.updateParentCurrentState = false; if(fn) fn(c);}
      copy.id = "0"
      while(currentVideoTargets.value[copy.id] !== undefined){
        copy.id = String(parseInt(copy.id) + 1);
      }
      currentVideoTargets.value[copy.id] = copy
    };

    const instance = {
      //state
      IS_config,
      currentConfig,
      current_state,
      loaded,
      renderAppMode,
      //targets
      transformedTargetCache,
      //video
      videoPlayers,
      //history
      interactionHistory,
      maxHistoryLength,
      //
      numVideoPlayers,
      lastUsedVideoPlayer,
      //computed
      targets,
      computeCurrentVideoTargets,
      currentVideoTargets,
      currentPage,
      //methods
      setupVideo,
      load,
      init,
      init_videoTargetCache,
      updateInteractionHistory,
      addInstanceLink,
      changeState,
      changeStateWithFrameNo,
      targetCacheModeChange,
      newVideoTarget,
      destroyVideoTarget,
      changeVideoFrame,
      createNewVideoPlayer,
      videoCacheModeChange,
      Up,
      Down,
      Delete,
      paste,
      //refs
      loomVideoCanvasRefs,
      videoPlayerRefs,
    };
    provide(`instance-${props.id}`, instance);
    return instance
  },
}
</script>

<style scoped>
  #loom {
    position: relative;
    width: 100%;
    height: 100%;
    margin: 0 auto;
  }
</style>