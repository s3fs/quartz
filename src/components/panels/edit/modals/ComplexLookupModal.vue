<script>
  import { usePreferenceStore } from '@/stores/preference'
  import { useConfigStore } from '@/stores/config'
  import { mapStores, mapState } from 'pinia'
  import { VueFinalModal } from 'vue-final-modal'

  import AuthTypeIcon from "@/components/panels/edit/fields/helpers/AuthTypeIcon.vue";


  import utilsNetwork from '@/lib/utils_network';


  export default {
    components: {
      VueFinalModal,
      AuthTypeIcon,

    },
    props: {
      // structure of the field that owns this modal
      structure: Object,
      // the inital search value starting the search
      searchValue: String
    },
    data() {
      return {

        modeSelect: null,
        searchValueLocal: null,


        searchTimeout: null,

        activeComplexSearch: [],
        activeComplexSearchInProgress: false,

        initalSearchState: true,


        activeContext: null


      }
    },
    computed: {
      // other computed properties
      // ...
      // gives access to this.counterStore and this.userStore
      ...mapStores(usePreferenceStore),
      // // // gives read access to this.count and this.double
      // ...mapState(usePreferenceStore, ['showPrefModal','showPrefModalGroup','styleDefault', 'showPrefModalGroup', 'fontFamilies']),

      // array of the pssobile groups from the stlyes

      ...mapState(useConfigStore, ['lookupConfig']),





      

      modalSelectOptions(){
        let options = []
        // add in the the defaul search ALL of everything possible
        //options.push({label: 'All', urls:null, processor:null})
        this.structure.valueConstraint.useValuesFrom.forEach((l)=>{
          if (this.lookupConfig[l]){
            this.lookupConfig[l].modes.forEach((mode)=>{              
              Object.keys(mode).forEach((k)=>{
                options.push({label: k, urls:mode[k].url, processor:this.lookupConfig[l].processor, minCharBeforeSearch: (this.lookupConfig[l].minCharBeforeSearch ? this.lookupConfig[l].minCharBeforeSearch : false), all:mode[k].all })
                // mark the first All one we find as the first one
                if (!this.modeSelect && mode[k].all){
                  this.modeSelect = k
                }
              })
            })
          }
        })        
        return options
      },
      modalSelectOptionsLabels(){
        return this.modalSelectOptions.map((o)=>{return o.label})
      },
    },

    watch: {


      // watching the search input, when it changes kick off a search
      searchValueLocal: async function(){
        this.doSearch()
      },


      modeSelect: async function(){
        this.doSearch()

      }



    },

    methods: {
      

      // watching the search input, when it changes kick off a search
      doSearch: async function(){

        if (!this.searchValueLocal){ return false}

        if (this.searchValueLocal.trim()==''){
          return false
        }

        if (this.searchValueLocal.length<3){
          // if it is non-latin 
          if (this.searchValueLocal.match(/[\u3040-\u30ff\u3400-\u4dbf\u4e00-\u9fff\uf900-\ufaff\uff66-\uff9f]/)){
            // if it is a CJK language don't impose that limit
          }else{
            // check the config, some vocabs have very short codes, like the marc geo
            // so if it is configed to allow short search overtide the < 3 rule
            let minCharBeforeSearch = 3
            this.modalSelectOptions.forEach((a)=>{
              if (a.minCharBeforeSearch && a.minCharBeforeSearch < minCharBeforeSearch){
                minCharBeforeSearch = a.minCharBeforeSearch
              }
            })

            if (this.searchValueLocal.length<minCharBeforeSearch){
              return false
            }
          }
        }
        window.clearTimeout(this.searchTimeout)

        let searchPayload = {
          processor: null,
          url: [],
          searchValueLocal: this.searchValueLocal
        }
        // if (this.modeSelect == 'All'){
        //   this.modalSelectOptions.forEach((a)=>{
        //     // use the ones in the config marked as "all" 
        //     if(a.all===true){
        //       searchPayload.processor=a.processor
        //       searchPayload.url.push(a.urls.replace('<QUERY>',this.searchValue))
        //     }
        //   })
          
        // }else{
          this.modalSelectOptions.forEach((a)=>{
            if (a.label==this.modeSelect){
              searchPayload.processor=a.processor
              searchPayload.url.push(a.urls.replace('<QUERY>',this.searchValueLocal))            
            }
          })


        this.searchTimeout = window.setTimeout(async ()=>{
          this.activeComplexSearchInProgress = true    
          this.activeComplexSearch = []      
          this.activeComplexSearch = await utilsNetwork.searchComplex(searchPayload)
          this.activeComplexSearchInProgress = false
          this.initalSearchState =false;
        }, 400)
        

      },



      inputKeydown: function(event){
        if (event.key==='ArrowDown'){
          this.$refs.selectOptions.focus()
          this.$refs.selectOptions.value=this.activeComplexSearch[0].uri
          this.selectChange()
        }
      },


      selectNav: function(event){
        if (event.target.selectedIndex == 0){
          // pop back up into the search field
          if (event.key==='ArrowUp'){
            if (this.$refs.inputLookup){
              this.$refs.inputLookup.focus()
              this.$nextTick(() => {
                window.setTimeout(()=> {
                  // put the cursor at the end
                  this.$refs.inputLookup.setSelectionRange(1000,1000);
                },10);             
              });
            }
          }          
        }
        if (event.key==='Enter' && event.shiftKey){
          console.log("emitComplexValue",this.activeContext)
          this.$emit('emitComplexValue', this.activeContext)
        }

      },    

      selectChange: async function(){


        let toLoad = this.activeComplexSearch[this.$refs.selectOptions.selectedIndex]

        console.log(toLoad)

        this.activeContext = {
            "contextValue": true,                  
            "source": [],
            "type": (toLoad.literal) ? "Literal Value" : null,
            "variant": [],
            "uri": (toLoad.literal) ? null : toLoad.uri,
            "title": toLoad.label,
            "contributor": [],
            "date": null,
            "genreForm": null,
            "nodeMap": {},
            "precoordinated" : false,
            "literal": true,
            "loading":true,
          }
       
        let results = await utilsNetwork.returnContext(toLoad.uri)
        results.loading = false

        // if this happens it means they selected something else very quickly
        // so don't go on and set it to this context, because its no longer the one they have selected
        if (toLoad.uri != this.activeComplexSearch[this.$refs.selectOptions.selectedIndex].uri){
          return false
        }


        this.activeContext = results

       
      },  

      rewriteURI: function(uri){

        if (!uri){
          return false
        }

        if (uri.includes('bibframe.example.org')){
          return false
        }

        if (uri.includes('/resources/hubs/') || uri.includes('/resources/works/') || uri.includes('/resources/instances/') || uri.includes('/resources/items/')){
          uri = uri.replace('https://id.loc.gov/', config.returnUrls().bfdb )
          uri = uri.replace('http://id.loc.gov/', config.returnUrls().bfdb )      
        }


        return uri
      },





    },

    updated: function(){

      this.$nextTick(() => {
        this.$nextTick(() => {
          if (this.$refs.inputLookup){
            this.$refs.inputLookup.focus()
          }

          this.searchValueLocal = this.searchValue

          let modalStopsAt = this.$refs.complexLookupModalContainer.getBoundingClientRect().height + this.$refs.complexLookupModalContainer.getBoundingClientRect().top
          let selectHeight =  modalStopsAt - this.$refs.selectOptions.getBoundingClientRect().y
          this.$refs.selectOptions.style.height = selectHeight - 1 + 'px'

          
          this.$refs.complexLookupModalDisplay.style.height = this.$refs.complexLookupModalContainer.getBoundingClientRect().height + 'px'

        })       
      })
    },

    mounted() {
      console.log("mounted yeah")


      if (this.modeSelect === null){
        this.modeSelect = this.modalSelectOptions[0].label
      }
      // this.$nextTick(()=>{
      //   this.loadPrefGroup()
      //   this.$nextTick(()=>{
      //     this.$refs.preferenceContent.style.height = this.initalHeight + 'px'
      //   })

      // })

    }
  }



</script>

<template>


    <VueFinalModal
      :hide-overlay="false"
      :overlay-transition="'vfm-fade'"
      :content-transition="'vfm-fade'"
      :click-to-close="true"
      :esc-to-close="true"
      :background="'non-interactive'"
      :lock-scroll="true"
      class="complex-lookup-modal"
      content-class="complex-lookup-modal-content"
      
      >

        <div ref="complexLookupModalContainer" class="complex-lookup-modal-container">

          <div class="complex-lookup-modal-container-parts">

            <div class="complex-lookup-modal-search"> 


              <template v-if="preferenceStore.returnValue('--b-edit-complex-use-select-dropdown') === false">
                <div class="toggle-btn-grp cssonly">
                  <div v-for="opt in modalSelectOptions"><input type="radio" :value="opt.label" class="search-mode-radio" v-model="modeSelect" name="searchMode"/><label onclick="" class="toggle-btn">{{opt.label}}</label></div>
                </div>
              </template>
              <template v-if="preferenceStore.returnValue('--b-edit-complex-use-select-dropdown') === true">
                <select v-model="modeSelect">
                  <option  v-for="opt in modalSelectOptions">{{opt.label}}</option>
                </select>
              </template>
              <input class="lookup-input" v-model="searchValueLocal" ref="inputLookup" @keydown="inputKeydown($event)" type="text" />

              <div>


                  <select size="100" ref="selectOptions" class="modal-entity-select" @change="selectChange($event)"  @keydown="selectNav($event)">
                    
                    <option v-if="activeComplexSearch.length == 0 && activeComplexSearchInProgress == false && initalSearchState != true">
                      No results found.
                    </option>
                    <option v-if="activeComplexSearchInProgress == true">
                      Searching...
                    </option>





                    <option v-for="(r,idx) in activeComplexSearch" :data-label="r.label" :value="r.uri" v-bind:key="idx" :style="(r.depreciated) ? 'color:red' : ''" class="complex-lookup-result" v-html="' ' + r.label + ((r.literal) ? ' [Literal]' : '')">
                    </option>
                  </select>              


              </div>



            </div>


            <div ref="complexLookupModalDisplay" class="complex-lookup-modal-display">
              
              <template v-if="activeContext !== null">

                  <h3><span class="modal-context-icon simptip-position-top" :data-tooltip="'Type: ' + activeContext.type"><AuthTypeIcon v-if="activeContext.type" :type="activeContext.type"></AuthTypeIcon></span>{{activeContext.title}}</h3>


                  <div class="complex-lookup-modal-display-type-buttons">
                    <div>
                        <div class="modal-context-data-title">{{activeContext.type}}</div>
                        <div v-if="activeContext.depreciated" style="background: pink;">
                          DEPRECIATED AUTHORITY
                        </div>
                        <a style="color:#2c3e50; float: none;    border: none;border-radius: 0;background-color: transparent;font-size: 1em;padding: 0;" :href="rewriteURI(activeContext.uri)" target="_blank">view on id.loc.gov</a>

                    </div>
                    <div class="complex-lookup-modal-display-buttons">
                      
                      <button @click="$emit('emitComplexValue', activeContext)">Add [Shift+Enter]</button>
                      <button @click="$emit('hideComplexModal')">Cancel [ESC]</button>

                    </div>


                  </div>

                  <div v-if="activeContext.variant && activeContext.variant.length>0">
                    <div class="modal-context-data-title modal-context-data-title-add-gap">Variants:</div>
                    <ul>
                      <li class="modal-context-data-li" v-for="(v,idx) in activeContext.variant" v-bind:key="'var' + idx">{{v}}</li>
                    </ul>


                  </div>

                  <div v-for="key in Object.keys(activeContext.nodeMap)" :key="key">
                    <div class="modal-context-data-title modal-context-data-title-add-gap">{{key}}:</div>
                      <ul>
                        <li class="modal-context-data-li" v-for="(v,idx) in activeContext.nodeMap[key]" v-bind:key="key+idx">{{v}}</li>
                      </ul>
                  </div>


                  <div v-if="activeContext.source && activeContext.source.length>0">
                    <div class="modal-context-data-title modal-context-data-title-add-gap">Sources:</div>
                    <ul>
                      <li class="modal-context-data-li" v-for="(v,idx) in activeContext.source" v-bind:key="'sources-'+idx">{{v}}</li>
                    </ul>


                  </div>

              </template>




              <div>
<!--                 <div class="load-wraper" style="height: 100px; width: 100px;">
                    <div class="activity" ></div>
                </div>      -->           

              </div>
            </div>

          </div>

        </div>

      
          
    </VueFinalModal>




</template>

<style>

.complex-lookup-modal-container{
  margin-left: auto;
  margin-right: auto;
  background-color: white;
  width: 85vw;
  height: 95vh;
}

.complex-lookup-modal-content{
  
}


@media all and (max-width: 1024px) {
  /* CSS rules here for screens lower than 750px */
  .complex-lookup-modal-container{

    width: 99vw;
    height: 95vh;
  }

}


</style>

<style scoped>

  .complex-lookup-modal-display-buttons{                      
    align-items: center;
    justify-content: center;
  }

  .complex-lookup-modal-display-buttons button{
    font-size: 1em;
    margin-right: 1em;
  }

  .complex-lookup-modal-display-type-buttons{
    display: flex;
  }
  .complex-lookup-modal-display-type-buttons div{
    flex:1;
  }
  .modal-context-data-title{
    font-weight: bold;
    
  }
  .modal-context-data-title-add-gap{
    margin-top: 1em;
  }
  .modal-context ul{
    margin-top: 0;
    margin-bottom: 0;
  }
  .modal-context-data-li{

  }

  h3{
    margin-bottom:1em;
  }

  .modal-context  h3{
    margin: 0;
    padding: 0;

  }



  .modal-entity-select{
    width: 100%;
    border:none;
    overflow-x: none;
    overflow-y: auto;
    outline:none;
  }
  .complex-lookup-results{
    padding: 0 1em 0 1em;
    height: 73%; 
    margin-top: 1.25em;

  }

  .lookup-input{
    width: 100%;
    margin: 3px;
    font-size: 1.5em;
  }

  .toggle-btn-grp div:focus{
    background-color: red;
  }

  .search-mode-radio:focus {
    background-color: red;
    outline: 1px solid black;
  }

  .complex-lookup-modal-container-parts{
    display: flex;
  }
  .complex-lookup-modal-container-parts div{
    flex:1;

  }
  .complex-lookup-modal-search{
    padding-right: 1em;
  }

  .complex-lookup-modal-display{
        overflow-y: auto;
        font-size: 1em;
  }
  label {

    font-size: 0.75em;
    white-space: nowrap;
  }

  .toggle-btn-grp {
    margin: 3px 0;
  }

  .toggle-btn {
    text-align: centre;
    padding: 0.1em 1em;
    color: #000;
    background-color: #FFF;
    border-radius: 10px;
    display: inline-block;
    border: solid 1px #CCC;
    cursor: pointer;
  }

  /* CSS only version */
  .toggle-btn-grp.cssonly * {
    width: 110px;
    height: 30px;
    line-height: 30px;
  }

  .toggle-btn-grp.cssonly div {
    display: inline-block;
    position: relative;
    margin: 5px 2px;
  }

  .toggle-btn-grp.cssonly div label {
    position: absolute;
    z-index: 0;
    padding: 0;
    text-align: center;
  }

  .toggle-btn-grp.cssonly div input {
    position: absolute;
    z-index: 1;
    cursor: pointer;
    opacity: 0;
  }

  .toggle-btn-grp.cssonly div:hover label {
    border: solid 1px #a0d5dc !important;
    background: #f1fdfe;
  }

  .toggle-btn-grp.cssonly div input:checked + label {
    background: lightskyblue;
    border: solid 1px blue !important;
  }


</style>
