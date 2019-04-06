<template>
  <div>
    <form class="pure-form pure-g">
      <input type="text" v-model="pageTitle" placeholder="App name" class="pure-u-11-12">
      <div class="pure-u-1-12" style="text-align: center">
        <img src="../assets/twitter.png" alt="Share on Twitter" v-on:click="shareOnTwitter()" style="width: 2em; height: 2em;">
      </div>
    </form>

    <div class="pure-g">
      <div class="pure-u-1">
        <codemirror v-model="script" v-on:change="onChangeScript()" :options="cmOptions" style="font-size: 1.2em;"></codemirror>
      </div>
    </div>
    <form class="pure-form pure-form-aligned">
      <label for="transpiler">Transpiler:</label>
      <select id="transpiler" v-model="transpiler" v-on:change="onChangeTranspiler()">
        <option v-for="t in transpilers" v-bind:value="t" >{{ t.name }}</option>
      </select>

      <label for="compression_alg">Compression:</label>
      <select id="compression_alg" v-model="compressionAlg" >
        <option v-for="a in compressionAlgs" v-bind:value="a" >{{ a.name }}</option>
      </select>

      <input type="checkbox" v-model="enableClickRun" v-on:change="setLocationHash()">: click_run
      <input type="checkbox" v-model="enablePromiseWait" v-on:change="setLocationHash()">: promise_wait

      <button v-if="enableClickRun" v-on:click="onClickClickRun()" class="pure-button" style="color: white; background: rgb(28, 184, 65)">
        {{ clickRunButtonText }}
      </button>
    </form>

    <form class="pure-form pure-g">
      <textarea v-model="inputText" placeholder="Input" rows="10" class="pure-u-1 pure-u-md-1-2"></textarea>
      <textarea v-model="outputText" placeholder="Output" rows="10" class="pure-u-1 pure-u-md-1-2"></textarea>
    </form>

    <button v-on:click="showError = !showError" v-bind:style="{color: hasError? 'red': ''}" class="pure-button">Show/Hide error</button>
    <span v-if="showError">
      <div class="pure-g">
        <textarea v-model="errorStr" class="pure-u-1"></textarea>
      </div>
    </span>

    <button v-on:click="showTranspiledJsCode = !showTranspiledJsCode" class="pure-button">Show/Hide transpiled JS code</button>
    <span v-if="showTranspiledJsCode">
      <div class="pure-g">
        <div class="pure-u-1">
          <codemirror v-model="transpiledJsCode" :options="{tabSize: 2, mode: 'text/javascript', lineNumbers: true, line: true, indentWithTabs: true}"></codemirror>
        </div>
      </div>
    </span>
  </div>
</template>

<script lang="ts">
import {Component, Prop, Vue, Watch} from 'vue-property-decorator';
import * as pako from 'pako';
import * as uaDeviceDetector from 'ua-device-detector';

import 'codemirror/lib/codemirror.css'
import 'codemirror/mode/ruby/ruby.js'
import 'codemirror/mode/javascript/javascript.js'

// Get Opal object
const Opal = (window as any).Opal;
// Get LZMA object
const LZMA = (window as any).LZMA;
// Get Babel
const Babel = (window as any).Babel;

type CompressionAlg = {
  name: string,
  compress: (raw: string) => string,
  decompress: (compressed: string) => string
}

const DeflateAlg: CompressionAlg = {
  name: "Deflate",
  compress: (str: string) => {
    // NOTE: Negative windowBits means no header and no checksum
    // (see: https://docs.python.org/3.6/library/zlib.html#zlib.decompress)
    const binStr = pako.deflate(str, {to: 'string', level: 9, windowBits: -8});
    return binStr;
  },
  decompress: (binStr: string) => {
    // NOTE: Negative windowBits means no header and no checksum
    // (see: https://docs.python.org/3.6/library/zlib.html#zlib.decompress)
    return pako.inflate(binStr, {to: 'string', windowBits: -8});
  }
};

const LZMAAlg: CompressionAlg = {
  name: "LZMA",
  compress: (str: string) => {
    const compressed: string = LZMA.compress(str, 9);
    // (from: https://github.com/alcor/itty-bitty/blob/5292c4b7891939dab89412f9e474bca707c9bec5/data.js#L25)
    // TODO: Not use any in Uint8Array
    // TODO: Not use any in apply
    return String.fromCharCode.apply(null, new Uint8Array(compressed as any) as any);
  },
  decompress: function(binStr) {
    return LZMA.decompress(binStr.split('').map(function(c){return c.charCodeAt(0)}));
  }
};

type Transpiler = {
  name: string,
  aceEditorMode: string,
  initLibrary: () => void,
  getExecutableFunctionAndTranspiledJsCode: (rubyScript: string) => { executableFunction: Function, transpiledJsCode: string }
};

// TODO: Move proper place
const RubyTranspiler: Transpiler = {
  name: "Ruby",
  aceEditorMode: "ruby",
  initLibrary: () => {
    Opal.load('opal');
    Opal.load('opal-parser');
  },
  getExecutableFunctionAndTranspiledJsCode: (rubyScript: string) => {
    // Use javascript global variable "INPUT"
    // (NOTE: `INPUT` will be pure JavaScript string variable)
    const rubyScriptWithInput = 's = `window.INPUT`\n' + rubyScript;
    // Transpile Ruby to JavaScript
    const transpiledJsCode =
      Opal.compile(rubyScriptWithInput)
    // Remove the first comment from transpiled code
          .replace(/\/\*.*\*\/\s*/, '');
    // Set executable function
    const executableFunction = new Function("return " + transpiledJsCode);
    return {
      executableFunction: executableFunction,
      transpiledJsCode: transpiledJsCode
    };
  }
};

const Es2017Transpiler: Transpiler = {
  name: "ES2017",
  aceEditorMode: "javascript",
  initLibrary: () => {},
  getExecutableFunctionAndTranspiledJsCode: (script) => {
    // Use javascript global variable "INPUT"
    // (NOTE: `INPUT` will be pure JavaScript string variable)
    const scriptWithInput = 'var s = window.INPUT;\n' + script;
    // Transpile
    const code = Babel.transform(scriptWithInput, {presets: ["es2017"]}).code;
    return {
      executableFunction: () => {
        return eval(code);
      },
      transpiledJsCode: code
    };
  }
};

const FuncEs2017Transpiler: Transpiler = {
  name: "ES2017 with Function",
  aceEditorMode: "javascript",
  initLibrary: () => {},
  getExecutableFunctionAndTranspiledJsCode: (script: string) => {
    // Use javascript global variable "INPUT"
    // (NOTE: `INPUT` will be pure JavaScript string variable)
    const scriptWithInput = 'var s = window.INPUT;\n' + script;
    // Transpile
    const code = Babel.transform(scriptWithInput, {presets: ["es2017"]}).code;
    // Generate executable function
    const executableFunction = new Function(code);
    return {
      executableFunction: executableFunction,
      transpiledJsCode: code
    };
  }
};


// Encode code
function encodeCode(code: string, compressor: (raw: string) => string) {
  try {
    const binStr = compressor(code);
    return btoa(binStr);
  } catch (err) {
    return "";
  }
}

// Decode code
function decodeCode(encodedCode: string, decompressor: (compressed: string) => string) {
  try {
    // Base64 => binary String
    const binStr = atob(encodedCode);
    return decompressor(binStr);
  } catch (err) {
    return "";
  }
}

// Parse location.hash and return page title and code
function parseLocationHash(): { pageTitle: string, urlOptions: string[], encodedCode: string } {
  // Split by "/"
  const splited = location.hash.split("/");
  if(splited.length >= 3) {
    // Get title
    const title = decodeURI(splited[0].substring(1).replace(/_/g, " "));
    // Get URL options
    const urlOptions = splited[1].split(",");
    // Get encoded code
    const encodedCode = splited.slice(2, splited.length).join("/");
    return {
      pageTitle: title,
      urlOptions: urlOptions,
      encodedCode: encodedCode
    };
  } else {
    return {
      pageTitle: "",
      urlOptions: [],
      encodedCode: ""
    }
  }
}

@Component
export default class Nipp extends Vue {
  compressionAlgs: ReadonlyArray<CompressionAlg> = [
    DeflateAlg,
    LZMAAlg
  ];
  // Compression algorithm
  compressionAlg: CompressionAlg = this.compressionAlgs[0];
  // Page title
  pageTitle = "";
  // Set empty string as default input
  inputText  = "";
  // Set empty string as default output
  outputText = "";
  // Script
  script = "";
  // Generated JavaScript code
  transpiledJsCode = "";
  // Whether transpiled JS code is shown or not
  showTranspiledJsCode = false;
  // Executable function which return result
  executableFunction: Function = () => {return "";};
  // Enable click-run or not
  // (click-run: Non-realtime/non-reactive evaluation)
  enableClickRun = false;
  // Use promise-wait or not
  enablePromiseWait = false;
  transpilers: ReadonlyArray<Transpiler> = [
    RubyTranspiler,
    Es2017Transpiler,
    FuncEs2017Transpiler
  ];
  // Set transpiler
  transpiler = this.transpilers[0];
  // Error string
  errorStr = "";
  // Whether error string is shown or not
  showError = true;
  // Whether has error or not
  hasError = false;
  // Text of click-run button
  clickRunButtonText = "";
  // Use textarea instead of ace
  useTextarea = false;

  mounted () {
    // Get page title and code
    const titleAndCode = parseLocationHash();
    if (titleAndCode.urlOptions.includes("lzma")) {
      this.compressionAlg = LZMAAlg;
    }
    // Set page title
    this.pageTitle = titleAndCode.pageTitle;
    // Set <title>
    document.title   = titleAndCode.pageTitle;
    // Set decoded location.hash as default script
    this.script = decodeCode(titleAndCode.encodedCode, this.compressionAlg.decompress);
    // Set enable-click-run
    this.enableClickRun = titleAndCode.urlOptions.includes("click_run");
    // Set enable-promise-wait
    this.enablePromiseWait = titleAndCode.urlOptions.includes("promise_wait");
    if (titleAndCode.urlOptions.includes("es2017")) {
      this.transpiler = Es2017Transpiler;
    } else if (titleAndCode.urlOptions.includes("func_es2017")) {
      this.transpiler = FuncEs2017Transpiler;
    }
    // Initialize library
    this.transpiler.initLibrary();
    // Set default value to global variable "INPUT"
    // TODO: duplicate code
    (window as any).INPUT = this.inputText;
    // Get device info
    const deviceInfo = uaDeviceDetector.parseUserAgent(window.navigator.userAgent);
    // Set click-run button text
    this.clickRunButtonText = "Run" + (deviceInfo.isDesktop() ? (deviceInfo.os === "mac"? "(⌘+Enter)" : "(Ctrl+Enter)") : "");
    // If enable click_run is disable
    if (!this.enableClickRun) {
      // Set default output
      this.setOutputText();
    }

    window.addEventListener('keydown', (e: WindowEventMap['keydown']) => {
      if((e.ctrlKey || e.metaKey) && e.key === "Enter") {
        Vue.nextTick(()=>{
          // Run onclick click-run
          this.onClickClickRun();
        })
      }
    });
  }

  @Watch("pageTitle")
  onChangePageTitle(): void {
    // Set page title
    document.title = this.pageTitle;
    // Set location.hash
    this.setLocationHash();
  }

  @Watch('compressionAlg')
  onChangeCompressionAlg() {
    // Update location.hash
    this.setLocationHash();
  }

  @Watch("script")
  onChangeScript(): void {
    // Set location.hash
    this.setLocationHash();
    // Transpile
    this.transpile();
  }

  // CodeMirror options
  get cmOptions() {
    const mode = this.transpiler === RubyTranspiler ? 'text/x-ruby': 'text/javascript';
    return {
      tabSize: 2,
      mode: mode,
      lineNumbers: true,
      line: true,
      indentWithTabs: true,
    };
  }

  setLocationHash() {
    // Create title part
    const titlePart = (this.pageTitle).replace(/ /g, "_");
    // Create options part
    const urlOptionsPart = this.getUrlOptionsPart();
    // Encode code
    const encodedCode = encodeCode(this.script, this.compressionAlg.compress);
    // Change location hash to the code
    location.hash = titlePart+"/"+urlOptionsPart+"/"+encodedCode;
  }

  // Generate options part
  getUrlOptionsPart(): string {
    const options: string[] = [];
    // (NOTE: transpiler:ruby is default so it should be pushed)
    if (this.transpiler === Es2017Transpiler) {
      options.push("es2017");
    } else if (this.transpiler === FuncEs2017Transpiler) {
      options.push("func_es2017");
    }
    // (NOTE: compression:deflate is default so it should be pushed)
    if (this.compressionAlg === LZMAAlg) {
      options.push("lzma");
    }
    // If click_run is enable
    if (this.enableClickRun) {
      options.push("click_run");
    }
    // If promise_wait is enable
    if (this.enablePromiseWait) {
      options.push("promise_wait");
    }
    // Generate options part
    return options.join(",");
  }

  // (NOTE: this is not typo. onclick "click_run")
  onClickClickRun() {
    // Set output text
    this.setOutputText();
  }

  onChangeTranspiler() {
    // Initialize library
    this.transpiler.initLibrary();
    // Ensure to call once
    this.transpiler.initLibrary = () => {};
    // Update location.hash
    this.setLocationHash();
    // Transpile
    this.transpile();
  }

  transpile() {
    try {
      // Transpile script and Set executable function
      const executableFunctionAndTraspiledJsCode = this.transpiler.getExecutableFunctionAndTranspiledJsCode(this.script);
      this.executableFunction = executableFunctionAndTraspiledJsCode.executableFunction;
      this.transpiledJsCode = executableFunctionAndTraspiledJsCode.transpiledJsCode;
      this.errorStr = "";
      this.hasError = false;
      // If enable click_run is disable
      if (!this.enableClickRun) {
        // Set output text
        this.setOutputText();
      }
    } catch (err) {
      console.log("Transpile compile", err);
      this.errorStr = err.toString();
      this.hasError = true;
    }
  }

  @Watch("inputText")
  onChangeInputText() {
    // If enable click_run is disable
    if (!this.enableClickRun) {
      // Set output text
      this.setOutputText();
    }
  }

  // Set output text
  setOutputText(){
    // Set global INPUT string variable
    (window as any).INPUT = this.inputText;
    try {
      // Get output
      const output = this.executableFunction();
      // If promise-wait is enable
      if(this.enablePromiseWait && output !== undefined) {
        // Get prototype of the object
        const proto = Object.getPrototypeOf(output);
        // If the output object is a Promise
        if(proto === Promise.prototype) {
          this.outputText = "<The promise is not complete yet>";
          output
            .then((res: any) => {
              Vue.nextTick(() => {
                this.outputText = res + "";
                // Set no error
                this.errorStr = "";
                this.hasError = false;
              });
            })
            .catch((err: Error) => {
              Vue.nextTick(()=>{
                this.outputText = "<Promise error: " + err.toString() + ">";
                this.errorStr = err.toString();
                this.hasError = true;
              });
            });
        }
      } else {
        // Set output text
        this.outputText = output + "";
        // Set no error
        this.errorStr = "";
        this.hasError = false;
      }
    } catch (err) {
      console.log("JS Runtime error", err);
      this.outputText = "";
      this.errorStr = err.toString();
      this.hasError = true;
    }
  }

  shareOnTwitter() {
    // (from: http://d.hatena.ne.jp/osyo-manga/20140717/1405626111)
    const url = 'https://twitter.com/share?text='+encodeURIComponent(this.pageTitle)+"&url=" + encodeURIComponent(location.href)+"&hashtags=nipp";
    window.open(url,'','scrollbars=yes,width=500,height=300,');
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
/* h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
} */
</style>