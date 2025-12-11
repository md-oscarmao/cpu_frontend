<template>
  <div class="y86-simulator">
    <h1>Y86-64 CPU 模拟器</h1>
    
    <!-- 文件上传区域 - 优化布局 -->
    <div class="file-section">
      <div class="file-upload-container">
        <input 
          type="file" 
          accept=".yo" 
          @change="handleFileUpload" 
          ref="fileInput"
          class="file-input"
        />
        <button @click="triggerFileUpload" class="upload-btn">
          {{ currentFile ? `已加载: ${currentFile.name}` : '选择 .yo 文件' }}
        </button>
      </div>
      <button @click="resetSimulator" class="reset-btn">重置</button>
      <div v-if="fileError" class="error-message">{{ fileError }}</div>
    </div>

    <!-- 进度条 - 修复圆圈滑块 -->
    <div class="progress-section">
      <div class="progress-header">
        <label>执行进度: {{ currentStep }} / {{ totalSteps }}</label>
        <span class="progress-status" :class="{'completed': isFinished}">
          {{ isFinished ? '执行完成' : '执行中' }}
        </span>
      </div>
      <div class="progress-bar-container">
        <div class="progress-background"></div>
        <div class="progress-fill" :style="{ width: progressPercentage + '%' }"></div>
        <div class="progress-thumb" :style="{ left: progressPercentage + '%' }">
          <div class="thumb-circle"></div>
          <div class="thumb-value">{{ currentStep }}</div>
        </div>
        <input 
          type="range" 
          min="0" 
          :max="totalSteps" 
          v-model="currentStep"
          @input="jumpToStep"
          class="progress-bar"
          :class="{'completed': isFinished}"
        />
      </div>
    </div>

    <!-- 执行状态 -->
    <div class="execution-status">
      <div v-if="isRunning" class="running-status">
        运行中... 已执行 {{ executedSteps }} 条指令
        <button @click="stopExecution" class="stop-btn">停止</button>
      </div>
      <div v-if="isFinished" class="finished-status">
        执行完成！总共执行了 {{ totalInstructions }} 条指令
      </div>
    </div>

    <!-- 控制按钮 -->
    <div class="control-buttons">
      <button @click="toggleRun" :disabled="!isLoaded" class="btn run-btn">
        {{ isRunning ? '停止' : 'Run' }}
      </button>
      <button @click="singleStep" :disabled="!isLoaded || isRunning || isFinished" class="btn next-instruction-btn">
        Next Instruction
      </button>
      <button @click="nextPhase" :disabled="!isLoaded || isRunning || isFinished || !showPhaseControls" class="btn next-phase-btn">
        Next Phase
      </button>
    </div>

    <!-- 主显示区域 -->
    <div class="main-display">
      <!-- 左侧：流水线寄存器阶段 -->
<div class="section pipeline-section">
  <h2>流水线寄存器阶段</h2>
  <div class="pipeline-stages">

    <!-- Fetch -->
    <div class="pipeline-stage" :class="{ active: currentPhase >= 0 }">
      <h3>Fetch (取指)</h3>
      <div v-if="pipelineState.fetch" class="stage-content">
        <p><strong>icode:</strong> {{ icodeDisplay(pipelineState.fetch.icode) }}</p>
        <p><strong>ifun:</strong> {{ pipelineState.fetch.ifun }}</p>
        <p><strong>rA:</strong> {{ regDisplay(pipelineState.fetch.rA) }}</p>
        <p><strong>rB:</strong> {{ regDisplay(pipelineState.fetch.rB) }}</p>
        <p><strong>valC:</strong> 0x{{ formatHex(pipelineState.fetch.valC, 8) }}</p>
        <p><strong>valP:</strong> 0x{{ formatHex(pipelineState.fetch.valP, 8) }}</p>
      </div>
    </div>

    <!-- Decode -->
    <div class="pipeline-stage" :class="{ active: currentPhase >= 1 }">
      <h3>Decode (译码)</h3>
      <div v-if="pipelineState.decode" class="stage-content">
        <p><strong>srcA:</strong> {{ regDisplay(pipelineState.decode.srcA) }}</p>
        <p><strong>srcB:</strong> {{ regDisplay(pipelineState.decode.srcB) }}</p>
        <p><strong>dstE:</strong> {{ regDisplay(pipelineState.decode.dstE) }}</p>
        <p><strong>dstM:</strong> {{ regDisplay(pipelineState.decode.dstM) }}</p>
        <p><strong>valA:</strong> 0x{{ formatHex(pipelineState.decode.valA, 8) }}</p>
        <p><strong>valB:</strong> 0x{{ formatHex(pipelineState.decode.valB, 8) }}</p>
      </div>
    </div>

    <!-- Execute -->
    <div class="pipeline-stage" :class="{ active: currentPhase >= 2 }">
      <h3>Execute (执行)</h3>
      <div v-if="pipelineState.execute" class="stage-content">
        <p><strong>valE:</strong> 0x{{ formatHex(pipelineState.execute.valE, 8) }}</p>
        <p><strong>Cnd:</strong> {{ pipelineState.execute.Cnd }}</p>
        <p><strong>aluA:</strong> 0x{{ formatHex(pipelineState.aluA, 8) }}</p>
        <p><strong>aluB:</strong> 0x{{ formatHex(pipelineState.aluB, 8) }}</p>
        <p><strong>alufun:</strong> {{ pipelineState.alufun }} ({{ getALUFunName(pipelineState.alufun) }})</p>
      </div>
    </div>

    <!-- Memory -->
    <div class="pipeline-stage" :class="{ active: currentPhase >= 3 }">
      <h3>Memory (访存)</h3>
      <div v-if="pipelineState.memory" class="stage-content">
        <p><strong>valM:</strong> 0x{{ formatHex(pipelineState.memory.valM, 8) }}</p>
        <p><strong>mem_addr:</strong> 0x{{ formatHex(pipelineState.memAddr, 8) }}</p>
        <p><strong>mem_data:</strong> 0x{{ formatHex(pipelineState.memData, 8) }}</p>
        <p><strong>read:</strong> {{ pipelineState.memRead }}</p>
        <p><strong>write:</strong> {{ pipelineState.memWrite }}</p>
      </div>
    </div>

    <!-- Write Back -->
    <div class="pipeline-stage" :class="{ active: currentPhase >= 4 }">
      <h3>Write Back (写回)</h3>
      <div v-if="pipelineState.writeback" class="stage-content">
        <p><strong>dstE:</strong> {{ regDisplay(pipelineState.decode.dstE) }} ← 0x{{ formatHex(pipelineState.execute.valE, 8) }}</p>
        <p><strong>dstM:</strong> {{ regDisplay(pipelineState.decode.dstM) }} ← 0x{{ formatHex(pipelineState.memory.valM, 8) }}</p>
      </div>
    </div>

    <!-- PC Update (新增) -->
    <div class="pipeline-stage" :class="{ active: currentPhase >= 5 }">
      <h3>PC Update (更新PC)</h3>
      <div v-if="pipelineState.pcUpdate" class="stage-content">
        <p><strong>newPC:</strong> 0x{{ formatHex(pipelineState.pcUpdate.newPC, 8) }}</p>
        <p><strong>predPC:</strong> 0x{{ formatHex(pipelineState.pcUpdate.predPC, 8) }}</p>
      </div>
    </div>

  </div>
</div>


      <!-- 中间：只保留内存指令视图 -->
      <div class="section instruction-section">
        <h2>当前指令</h2>
        <div class="memory-view">
          <h3>内存指令视图</h3>
          <div class="memory-table">
            <div class="memory-row header">
              <div class="mem-addr">地址</div>
              <div class="mem-bytes">机器码</div>
              <div class="mem-asm">反汇编</div>
            </div>
            <div 
              v-for="(instr, idx) in disassemblyView" 
              :key="idx"
              class="memory-row"
              :class="{ 'current-pc': instr.pc === cpuState.pc }"
            >
              <div class="mem-addr">0x{{ formatHex(instr.pc, 8) }}</div>
              <div class="mem-bytes">
                <span v-for="(byte, bIdx) in instr.bytes" :key="bIdx" class="byte">
                  {{ formatHex(byte, 2) }}
                </span>
              </div>
              <div class="mem-asm">{{ instr.disasm }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- 右侧：CPU状态 - 优化寄存器显示 -->
      <div class="section cpu-state-section">
        <h2>CPU 状态</h2>
        
        <div class="registers">
          <h3>寄存器</h3>
          <div class="register-grid">
            <div v-for="reg in registers" :key="reg.name" class="register-item">
              <span class="reg-name">{{ reg.name }}</span>
              <span class="reg-value">0x{{ formatHex(reg.value, 6) }}</span>
              <span v-if="reg.changed" class="reg-changed">↻</span>
            </div>
          </div>
        </div>

        <div class="condition-codes">
          <h3>条件码</h3>
          <div class="cc-flags">
            <div class="cc-flag" :class="{ active: cpuState.cc.ZF }">
              <span>ZF</span>
              <span class="cc-value">{{ cpuState.cc.ZF ? 1 : 0 }}</span>
            </div>
            <div class="cc-flag" :class="{ active: cpuState.cc.SF }">
              <span>SF</span>
              <span class="cc-value">{{ cpuState.cc.SF ? 1 : 0 }}</span>
            </div>
            <div class="cc-flag" :class="{ active: cpuState.cc.OF }">
              <span>OF</span>
              <span class="cc-value">{{ cpuState.cc.OF ? 1 : 0 }}</span>
            </div>
          </div>
        </div>

        <div class="status-section">
          <h3>状态</h3>
          <div class="status-display" :class="getStatClass(cpuState.stat)">
            {{ getStatName(cpuState.stat) }}
          </div>
        </div>

        <div class="program-counter">
          <h3>程序计数器</h3>
          <div class="pc-value">
            0x{{ formatHex(cpuState.pc, 8) }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
// JavaScript代码保持不变
export default {
  name: 'Y86Simulator',
  data() {
    return {
      // 模拟器状态
      cpuState: {
        pc: 0,
        regs: new Array(15).fill(0),
        cc: { ZF: 1, SF: 0, OF: 0 },
        stat: 0,
        mem: new Array(0x100000).fill(0)
      },
      
      // 流水线寄存器状态
      pipelineState: {
        fetch: null,
        decode: null,
        execute: null,
        memory: null,
        writeback: null,
        aluA: 0,
        aluB: 0,
        alufun: 0,
        memAddr: 0,
        memData: 0,
        memRead: false,
        memWrite: false
      },
      
      // 当前指令信息
      currentInstruction: null,
      currentDisassembly: '',
      
      // 控制状态
      currentStep: 0,
      totalSteps: 0,
      currentPhase: -1,
      currentCycle: 0,
      executedSteps: 0,
      totalInstructions: 0,
      
      // 文件处理
      currentFile: null,
      fileError: null,
      isLoaded: false,
      isRunning: false,
      isFinished: false,
      showPhaseControls: false,
      
      // 执行控制
      executionTimer: null,
      
      // 执行历史
      executionHistory: [],
      
      // 寄存器名称映射
      regNames: [
        'rax', 'rcx', 'rdx', 'rbx', 'rsp', 'rbp', 'rsi', 'rdi',
        'r8', 'r9', 'r10', 'r11', 'r12', 'r13', 'r14', 'NONE'
      ],
      
      // 指令名称映射
      icodeNames: [
      'halt',   // 0
      'nop',    // 1
      'rrmovq', // 2
      'irmovq', // 3
      'rmmovq', // 4
      'mrmovq', // 5
      'OPq',    // 6 （真正解释需看 ifun）
      'jXX',    // 7
      'call',   // 8
      'ret',    // 9
      'pushq',  // 10
      'popq',   // 11
      'prt',    // 12
      'unused', // 13
      'unused', // 14
      'unused'  // 15
      ],

      
      // ALU操作名称
      aluFunNames: ['add', 'sub', 'and', 'xor'],
      
      // 状态名称
      statNames: ['SAOK', 'SHLT', 'SADR', 'SINS'],
      
      // 阶段名称
      phaseNames: ['取指', '译码', '执行', '访存', '写回', 'PC更新'],
      
      // 指令缓存
      instructionCache: new Map(),
      
      // 性能控制
      executionSpeed: 10,
      maxSteps: 10000
    };
  },
  
  computed: {
    registers() {
      return this.regNames.slice(0, 15).map((name, idx) => ({
        name,
        value: this.cpuState.regs[idx],
        changed: this.executionHistory.length > 0 && 
                 this.executionHistory[this.executionHistory.length - 1].regChanges?.includes(idx)
      }));
    },
    
    disassemblyView() {
      const instructions = [];
      let pc = this.cpuState.pc;
      
      for (let i = 0; i < 10 && pc < this.cpuState.mem.length; i++) {
        const instr = this.disassembleAt(pc);
        if (!instr) break;
        
        instructions.push({
          pc,
          bytes: instr.bytes,
          disasm: instr.disasm
        });
        
        pc += instr.length;
      }
      
      return instructions;
    },
    
    progressPercentage() {
      if (this.totalSteps === 0) return 0;
      return Math.min(100, (this.currentStep / this.totalSteps) * 100);
    }
  },
  
  methods: {
    // 格式化十六进制 - 支持不同的位数
    formatHex(value, digits) {
      if (typeof value === 'bigint') {
        // 对于寄存器值显示6位，其他保持原样
        if (digits === 6) {
          // 取低6位十六进制数
          const hexStr = value.toString(16);
          return hexStr.slice(-6).padStart(6, '0');
        }
        return value.toString(16).padStart(digits, '0');
      }
      return value.toString(16).padStart(digits, '0');
    },
    regDisplay(id) {
      return `${id} (${this.regNames[id]})`;
    },

    icodeDisplay(code) {
      return `${code} (${this.icodeNames[code]})`;
    },

    // 触发文件上传
    triggerFileUpload() {
      this.$refs.fileInput.click();
    },
    
    // 处理文件上传
    async handleFileUpload(event) {
      const file = event.target.files[0];
      if (!file) return;
      
      this.currentFile = file;
      this.fileError = null;
      
      try {
        const content = await this.readFile(file);
        this.loadYoFile(content);
        this.isLoaded = true;
        this.resetExecution();
        this.showPhaseControls = true;
        this.instructionCache.clear();
      } catch (error) {
        console.error('加载文件失败:', error);
        this.fileError = `加载文件失败: ${error.message}`;
      }
    },
    
    // 读取文件内容
    readFile(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = (e) => resolve(e.target.result);
        reader.onerror = () => reject(new Error('读取文件失败'));
        reader.readAsText(file);
      });
    },
    
    // 加载.yo文件
    loadYoFile(content) {
      // 清空内存
      this.cpuState.mem = new Array(0x100000).fill(0);
      
      const lines = content.split('\n');
      let hasValidData = false;
      
      lines.forEach((line, lineNum) => {
        line = line.trim();
        if (!line || line.startsWith('#') || line === '|') return;
        
        const commentIndex = line.indexOf('|');
        if (commentIndex !== -1) {
          line = line.substring(0, commentIndex).trim();
        }
        
        const colonIndex = line.indexOf(':');
        if (colonIndex === -1) return;
        
        const addrStr = line.substring(0, colonIndex).trim();
        const dataStr = line.substring(colonIndex + 1).trim();
        
        let addr;
        if (addrStr.startsWith('0x')) {
          addr = parseInt(addrStr.substring(2), 16);
        } else {
          addr = parseInt(addrStr, 16);
        }
        
        if (isNaN(addr)) {
          console.warn(`第${lineNum + 1}行: 无效地址 "${addrStr}"`);
          return;
        }
        
        const bytes = [];
        const hexStr = dataStr.replace(/\s+/g, '');
        
        for (let i = 0; i < hexStr.length; i += 2) {
          if (i + 1 >= hexStr.length) break;
          
          const byteStr = hexStr.substring(i, i + 2);
          const byte = parseInt(byteStr, 16);
          
          if (isNaN(byte)) {
            console.warn(`第${lineNum + 1}行: 无效字节 "${byteStr}"`);
            continue;
          }
          
          bytes.push(byte);
        }
        
        if (bytes.length > 0) {
          bytes.forEach((byte, idx) => {
            if (addr + idx < this.cpuState.mem.length) {
              this.cpuState.mem[addr + idx] = byte;
            }
          });
          hasValidData = true;
        }
      });
      
      if (!hasValidData) {
        throw new Error('未找到有效的.yo文件数据');
      }
      
      this.cpuState.pc = 0;
    },
    
    // 重置模拟器
    resetSimulator() {
      this.stopExecution();
      
      this.cpuState = {
        pc: 0,
        regs: new Array(15).fill(0),
        cc: { ZF: 1, SF: 0, OF: 0 },
        stat: 0,
        mem: new Array(0x100000).fill(0)
      };
      
      this.pipelineState = {
        fetch: null,
        decode: null,
        execute: null,
        memory: null,
        writeback: null,
        aluA: 0,
        aluB: 0,
        alufun: 0,
        memAddr: 0,
        memData: 0,
        memRead: false,
        memWrite: false
      };
      
      this.currentInstruction = null;
      this.currentStep = 0;
      this.totalSteps = 0;
      this.currentPhase = -1;
      this.currentCycle = 0;
      this.executedSteps = 0;
      this.totalInstructions = 0;
      this.executionHistory = [];
      this.isLoaded = false;
      this.isRunning = false;
      this.isFinished = false;
      this.showPhaseControls = false;
      this.fileError = null;
      this.instructionCache.clear();
      
      if (this.currentFile) {
        this.handleFileUpload({ target: { files: [this.currentFile] } });
      }
    },
    
    // 重置执行状态
    resetExecution() {
      this.stopExecution();
      this.currentStep = 0;
      this.totalSteps = 0;
      this.currentPhase = -1;
      this.currentCycle = 0;
      this.executedSteps = 0;
      this.totalInstructions = 0;
      this.executionHistory = [];
      this.isFinished = false;
      this.instructionCache.clear();
    },
    
    // 切换运行状态
    toggleRun() {
      if (this.isRunning) {
        this.stopExecution();
      } else {
        this.runToCompletion();
      }
    },
    
    // 运行到完成 - 使用异步执行
    async runToCompletion() {
      if (this.isRunning || this.isFinished) return;
      
      this.isRunning = true;
      this.isFinished = false;
      
      const runAsync = async () => {
        let stepsExecuted = 0;
        
        while (this.isRunning && this.cpuState.stat === 0 && stepsExecuted < this.maxSteps) {
          // 保存当前状态
          this.saveCurrentState();
          
          // 执行一条完整指令
          for (let phase = 0; phase < 6; phase++) {
            this.currentPhase = phase;
            this.executePhase(phase);
            
            // 短暂的延迟以更新UI
            if (phase === 5) { // PC更新阶段后
              await this.delay(50);
            }
          }
          
          this.currentPhase = -1;
          this.currentCycle++;
          this.totalInstructions++;
          this.executedSteps++;
          stepsExecuted++;
          
          // 更新进度
          this.currentStep = this.executionHistory.length;
          this.totalSteps = this.executionHistory.length;
          
          // 检查是否应该停止
          if (this.cpuState.stat !== 0) {
            this.isFinished = true;
            this.isRunning = false;
            // 确保进度条到终点
            this.currentStep = this.totalSteps;
            break;
          }
        }
        
        // 如果达到最大步数，自动停止
        if (stepsExecuted >= this.maxSteps) {
          console.warn(`达到最大执行步数 ${this.maxSteps}，自动停止`);
          this.isRunning = false;
          // 确保进度条到终点
          this.currentStep = this.totalSteps;
        }
      };
      
      // 开始执行
      runAsync().catch(error => {
        console.error('执行出错:', error);
        this.isRunning = false;
      });
    },
    
    // 延迟函数
    delay(ms) {
      return new Promise(resolve => setTimeout(resolve, ms));
    },
    
    // 停止执行
    stopExecution() {
      this.isRunning = false;
      if (this.executionTimer) {
        clearTimeout(this.executionTimer);
        this.executionTimer = null;
      }
    },
    
    // 单步执行
    singleStep() {
      if (this.isRunning || this.isFinished) return;
      
      // 保存当前状态
      this.saveCurrentState();
      
      // 执行一条完整指令
      for (let phase = 0; phase < 6; phase++) {
        this.currentPhase = phase;
        this.executePhase(phase);
      }
      
      this.currentPhase = -1;
      this.currentCycle++;
      this.totalInstructions++;
      this.executedSteps++;
      
      // 检查是否结束
      if (this.cpuState.stat !== 0) {
        this.isFinished = true;
        // 确保进度条到终点
        this.currentStep = this.totalSteps;
      }
      
      this.currentStep = this.executionHistory.length;
      this.totalSteps = this.executionHistory.length;
    },
    
    // 执行下一个阶段
    nextPhase() {
      if (this.isRunning || this.isFinished) return;
      
      if (this.currentPhase === -1) {
        this.saveCurrentState();
      }
      
      this.currentPhase = (this.currentPhase + 1) % 6;
      this.executePhase(this.currentPhase);
      
      if (this.currentPhase === 5) {
        this.currentCycle++;
        this.totalInstructions++;
        this.executedSteps++;
        this.currentPhase = -1;
        
        if (this.cpuState.stat !== 0) {
          this.isFinished = true;
          // 确保进度条到终点
          this.currentStep = this.totalSteps;
        }
        
        this.currentStep = this.executionHistory.length;
        this.totalSteps = this.executionHistory.length;
      }
    },
    
    // 执行特定阶段
    executePhase(phase) {
      try {
        switch (phase) {
          case 0: // 取指
            this.fetchStage();
            break;
          case 1: // 译码
            this.decodeStage();
            break;
          case 2: // 执行
            this.executeStage();
            break;
          case 3: // 访存
            this.memoryStage();
            break;
          case 4: // 写回
            this.writebackStage();
            break;
          case 5: // PC更新
            this.pcUpdateStage();
            break;
        }
      } catch (error) {
        console.error(`执行阶段 ${phase} 出错:`, error);
        this.cpuState.stat = 3; // SINS - 非法指令
      }
    },
    
    // 取指阶段
    fetchStage() {
      const pc = this.cpuState.pc;
      
      // 读取第一个字节（icode:ifun）
      const byte0 = this.readMemByte(pc);
      const icode = (byte0 >> 4) & 0xF;
      const ifun = byte0 & 0xF;
      
      // 检查指令是否合法
      const instrValid = icode >= 0 && icode <= 15;
      
      // 确定是否需要寄存器字节和立即数
      const needRegids = [2, 3, 4, 5, 6, 10, 11, 12].includes(icode);
      const needValC = [3, 4, 5, 7, 8].includes(icode);
      
      // 计算指令长度
      let length = 1;
      if (needRegids) length++;
      if (needValC) length += 8;
      
      // 计算valP
      const valP = pc + length;
      
      // 读取寄存器字节（如果需要）
      let rA = 15, rB = 15;
      let valC = 0n;
      
      if (needRegids) {
        const regbyte = this.readMemByte(pc + 1);
        rA = (regbyte >> 4) & 0xF;
        rB = regbyte & 0xF;
      }
      
      // 读取立即数（如果需要）
      if (needValC) {
        const valCAddr = pc + (needRegids ? 2 : 1);
        valC = this.readMemWord(valCAddr);
      }
      
      // 获取机器码字符串
      const machineCode = this.getMachineCodeString(pc, length);
      
      // 更新流水线寄存器
      this.pipelineState.fetch = {
        icode, ifun, rA, rB, valC, valP,
        needRegids, needValC, imem_error: false, instr_valid: instrValid
      };
      
      // 设置当前指令
      this.currentInstruction = {
        pc,
        icode,
        ifun,
        rA,
        rB,
        valC,
        length,
        machineCode,
        name: this.getInstructionName(icode, ifun),
        needRegids,
        needValC
      };
      
      // 生成反汇编
      this.generateDisassembly();
    },
    
    // 译码阶段
    decodeStage() {
      if (!this.pipelineState.fetch) return;
      
      const F = this.pipelineState.fetch;
      const icode = F.icode;
      const rA = F.rA;
      const rB = F.rB;
      
      // 选择源寄存器
      let srcA = 15, srcB = 15;
      
      switch (icode) {
        case 2: // rrmovq
        case 4: // rmmovq
        case 6: // OPq
        case 10: // pushq
        case 12: // prt
          srcA = rA;
          break;
        case 11: // popq
        case 9: // ret
          srcA = 4; // rsp
          break;
      }
      
      switch (icode) {
        case 4: // rmmovq
        case 5: // mrmovq
        case 6: // OPq
          srcB = rB;
          break;
        case 8: // call
        case 9: // ret
        case 10: // pushq
        case 11: // popq
          srcB = 4; // rsp
          break;
      }
      
      // 读取寄存器值
      const valA = this.readReg(srcA);
      const valB = this.readReg(srcB);
      
      // 对于call/jmp，valA需要特殊处理
      if (icode === 8 || icode === 7) {
        // 在compute_valA中处理
      }
      
      // 选择目的寄存器
      let dstE = 15, dstM = 15;
      
      switch (icode) {
        case 2: // rrmovq
        case 3: // irmovq
        case 6: // OPq
          dstE = rB;
          break;
        case 8: // call
        case 9: // ret
        case 10: // pushq
        case 11: // popq
          dstE = 4; // rsp
          break;
      }
      
      switch (icode) {
        case 5: // mrmovq
        case 11: // popq
          dstM = rA;
          break;
      }
      
      this.pipelineState.decode = { srcA, srcB, dstE, dstM, valA, valB };
    },
    
    // 执行阶段
    executeStage() {
      if (!this.pipelineState.fetch || !this.pipelineState.decode) return;
      
      const F = this.pipelineState.fetch;
      const D = this.pipelineState.decode;
      
      // 选择ALU输入
      let aluA = 0n, aluB = 0n;
      
      switch (F.icode) {
        case 2: // rrmovq
        case 6: // OPq
          aluA = D.valA;
          break;
        case 3: // irmovq
        case 4: // rmmovq
        case 5: // mrmovq
          aluA = F.valC;
          break;
        case 8: // call
        case 10: // pushq
          aluA = -8n;
          break;
        case 9: // ret
        case 11: // popq
          aluA = 8n;
          break;
      }
      
      switch (F.icode) {
        case 4: // rmmovq
        case 5: // mrmovq
        case 6: // OPq
        case 8: // call
        case 9: // ret
        case 10: // pushq
        case 11: // popq
          aluB = D.valB;
          break;
      }
      
      // 选择ALU操作
      let alufun = 0; // 默认加法
      if (F.icode === 6) {
        alufun = F.ifun;
      }
      
      // 执行ALU操作
      let valE = 0n;
      switch (alufun) {
        case 0: // add
          valE = aluB + aluA;
          break;
        case 1: // sub
          valE = aluB - aluA;
          break;
        case 2: // and
          valE = aluB & aluA;
          break;
        case 3: // xor
          valE = aluB ^ aluA;
          break;
        default:
          valE = aluB + aluA;
      }
      
      // 更新条件码（仅OPq指令）
      if (F.icode === 6) {
        this.cpuState.cc.ZF = (valE === 0n);
        this.cpuState.cc.SF = (valE < 0n);
        this.cpuState.cc.OF = false; // 简化溢出检测
      }
      
      // 计算条件
      let Cnd = true;
      if (F.icode === 2 || F.icode === 7) {
        Cnd = this.computeCondition(F.ifun);
      }
      
      this.pipelineState.aluA = aluA;
      this.pipelineState.aluB = aluB;
      this.pipelineState.alufun = alufun;
      this.pipelineState.execute = { valE, Cnd };
    },
    
    // 访存阶段
    memoryStage() {
      if (!this.pipelineState.fetch || !this.pipelineState.decode || !this.pipelineState.execute) return;
      
      const F = this.pipelineState.fetch;
      const D = this.pipelineState.decode;
      const E = this.pipelineState.execute;
      
      // 选择内存地址
      let memAddr = 0n;
      switch (F.icode) {
        case 4: // rmmovq
        case 5: // mrmovq
        case 8: // call
        case 10: // pushq
          memAddr = E.valE;
          break;
        case 9: // ret
        case 11: // popq
          memAddr = D.valA;
          break;
      }
      
      // 选择写入数据
      let memData = 0n;
      let memRead = false;
      let memWrite = false;
      let valM = 0n;
      
      switch (F.icode) {
        case 5: // mrmovq
        case 11: // popq
        case 9: // ret
          memRead = true;
          valM = this.readMemWord(Number(memAddr));
          break;
        case 4: // rmmovq
        case 10: // pushq
          memWrite = true;
          memData = D.valA;
          this.writeMemWord(Number(memAddr), memData);
          break;
        case 8: // call
          memWrite = true;
          memData = BigInt(F.valP);
          this.writeMemWord(Number(memAddr), memData);
          break;
      }
      
      this.pipelineState.memory = { valM, dmem_error: false };
      this.pipelineState.memAddr = memAddr;
      this.pipelineState.memData = memData;
      this.pipelineState.memRead = memRead;
      this.pipelineState.memWrite = memWrite;
    },
    
    // 写回阶段
    writebackStage() {
      if (!this.pipelineState.decode || !this.pipelineState.execute || !this.pipelineState.memory) return;
      
      const D = this.pipelineState.decode;
      const E = this.pipelineState.execute;
      const M = this.pipelineState.memory;
      const F = this.pipelineState.fetch;
      
      // 记录寄存器变化
      const regChanges = [];
      
      // 写入dstE
      if (D.dstE !== 15) {
        // 对于条件传送指令，需要检查条件
        if (F.icode === 2 && !E.Cnd) {
          // 条件不满足，不写入
        } else {
          this.writeReg(D.dstE, E.valE);
          regChanges.push(D.dstE);
        }
      }
      
      // 写入dstM
      if (D.dstM !== 15) {
        this.writeReg(D.dstM, M.valM);
        regChanges.push(D.dstM);
      }
      
      // 保存寄存器变化到历史记录
      if (this.executionHistory.length > 0) {
        this.executionHistory[this.executionHistory.length - 1].regChanges = regChanges;
      }
      
      this.pipelineState.writeback = { done: true };
    },
    
    // PC更新阶段
    pcUpdateStage() {
      if (!this.pipelineState.fetch || !this.pipelineState.execute || !this.pipelineState.memory) return;
      
      const F = this.pipelineState.fetch;
      const E = this.pipelineState.execute;
      const M = this.pipelineState.memory;
      
      // 根据Y86规范计算新PC
      let newPC = F.valP; // 默认顺序执行
      
      switch (F.icode) {
        case 7: // jmp (IJXX)
          if (E.Cnd) {
            newPC = Number(F.valC);
          }
          break;
        case 8: // call (ICALL)
          newPC = Number(F.valC);
          break;
        case 9: // ret (IRET)
          newPC = Number(M.valM);
          break;
        case 0: // halt (IHALT)
          this.cpuState.stat = 1; // SHLT
          break;
      }
      
      // 更新PC
      this.cpuState.pc = newPC;
      
      // 检查prt指令并输出
      if (F.icode === 12 && F.ifun === 0) {
        console.log(`prt输出: ${this.pipelineState.decode.valA}`);
      }
      
      // 只有halt指令才会设置状态为SHLT，其他指令保持SAOK
      if (F.icode !== 0) {
        this.cpuState.stat = 0; // SAOK
      }
    },
    
    // 保存当前状态到历史记录
    saveCurrentState() {
      // 创建状态快照，处理BigInt序列化问题
      const snapshot = {
        pc: this.cpuState.pc,
        regs: this.cpuState.regs.map(val => 
          typeof val === 'bigint' ? val.toString() : val
        ),
        cc: { ...this.cpuState.cc },
        stat: this.cpuState.stat,
        pipeline: this.clonePipelineState(this.pipelineState),
        instruction: this.currentInstruction ? {
          ...this.currentInstruction,
          valC: typeof this.currentInstruction.valC === 'bigint' 
            ? this.currentInstruction.valC.toString() 
            : this.currentInstruction.valC,
          pc: this.currentInstruction.pc
        } : null,
        cycle: this.currentCycle,
        step: this.currentStep
      };
      
      this.executionHistory.push(snapshot);
    },
    
    // 克隆流水线状态，处理BigInt
    clonePipelineState(state) {
      if (!state) return null;
      
      const cloned = {};
      
      // 克隆基本属性
      for (const key in state) {
        if (key === 'fetch' && state.fetch) {
          cloned.fetch = { ...state.fetch };
          if (typeof state.fetch.valC === 'bigint') {
            cloned.fetch.valC = state.fetch.valC.toString();
          }
        } else if (key === 'decode' && state.decode) {
          cloned.decode = { ...state.decode };
          if (typeof state.decode.valA === 'bigint') {
            cloned.decode.valA = state.decode.valA.toString();
          }
          if (typeof state.decode.valB === 'bigint') {
            cloned.decode.valB = state.decode.valB.toString();
          }
        } else if (key === 'execute' && state.execute) {
          cloned.execute = { ...state.execute };
          if (typeof state.execute.valE === 'bigint') {
            cloned.execute.valE = state.execute.valE.toString();
          }
        } else if (key === 'memory' && state.memory) {
          cloned.memory = { ...state.memory };
          if (typeof state.memory.valM === 'bigint') {
            cloned.memory.valM = state.memory.valM.toString();
          }
        } else if (key === 'aluA' && typeof state.aluA === 'bigint') {
          cloned.aluA = state.aluA.toString();
        } else if (key === 'aluB' && typeof state.aluB === 'bigint') {
          cloned.aluB = state.aluB.toString();
        } else if (key === 'memAddr' && typeof state.memAddr === 'bigint') {
          cloned.memAddr = state.memAddr.toString();
        } else if (key === 'memData' && typeof state.memData === 'bigint') {
          cloned.memData = state.memData.toString();
        } else {
          cloned[key] = state[key];
        }
      }
      
      return cloned;
    },
    
    // 跳转到指定步骤
    jumpToStep() {
      if (this.currentStep < 0 || this.currentStep >= this.executionHistory.length) return;
      
      const snapshot = this.executionHistory[this.currentStep];
      
      // 恢复CPU状态
      this.cpuState.pc = snapshot.pc;
      this.cpuState.regs = snapshot.regs.map(val => 
        typeof val === 'string' ? BigInt(val) : (typeof val === 'bigint' ? val : BigInt(val || 0))
      );
      this.cpuState.cc = { ...snapshot.cc };
      this.cpuState.stat = snapshot.stat;
      
      // 恢复流水线状态
      this.pipelineState = this.restorePipelineState(snapshot.pipeline);
      
      // 恢复当前指令
      this.currentInstruction = snapshot.instruction ? {
        ...snapshot.instruction,
        valC: typeof snapshot.instruction.valC === 'string' 
          ? BigInt(snapshot.instruction.valC) 
          : snapshot.instruction.valC,
        pc: snapshot.instruction.pc
      } : null;
      
      this.currentCycle = snapshot.cycle;
      
      // 生成反汇编
      if (this.currentInstruction) {
        this.generateDisassembly();
      }
      
      // 更新阶段显示
      if (this.pipelineState.fetch) {
        this.currentPhase = 0;
        if (this.pipelineState.decode) this.currentPhase = 1;
        if (this.pipelineState.execute) this.currentPhase = 2;
        if (this.pipelineState.memory) this.currentPhase = 3;
        if (this.pipelineState.writeback) this.currentPhase = 4;
      } else {
        this.currentPhase = -1;
      }
    },
    
    // 恢复流水线状态，处理BigInt
    restorePipelineState(state) {
      if (!state) return {
        fetch: null,
        decode: null,
        execute: null,
        memory: null,
        writeback: null,
        aluA: 0,
        aluB: 0,
        alufun: 0,
        memAddr: 0,
        memData: 0,
        memRead: false,
        memWrite: false
      };
      
      const restored = {};
      
      // 恢复基本属性
      for (const key in state) {
        if (key === 'fetch' && state.fetch) {
          restored.fetch = { ...state.fetch };
          if (typeof state.fetch.valC === 'string') {
            restored.fetch.valC = BigInt(state.fetch.valC);
          }
        } else if (key === 'decode' && state.decode) {
          restored.decode = { ...state.decode };
          if (typeof state.decode.valA === 'string') {
            restored.decode.valA = BigInt(state.decode.valA);
          }
          if (typeof state.decode.valB === 'string') {
            restored.decode.valB = BigInt(state.decode.valB);
          }
        } else if (key === 'execute' && state.execute) {
          restored.execute = { ...state.execute };
          if (typeof state.execute.valE === 'string') {
            restored.execute.valE = BigInt(state.execute.valE);
          }
        } else if (key === 'memory' && state.memory) {
          restored.memory = { ...state.memory };
          if (typeof state.memory.valM === 'string') {
            restored.memory.valM = BigInt(state.memory.valM);
          }
        } else if (key === 'aluA' && typeof state.aluA === 'string') {
          restored.aluA = BigInt(state.aluA);
        } else if (key === 'aluB' && typeof state.aluB === 'string') {
          restored.aluB = BigInt(state.aluB);
        } else if (key === 'memAddr' && typeof state.memAddr === 'string') {
          restored.memAddr = BigInt(state.memAddr);
        } else if (key === 'memData' && typeof state.memData === 'string') {
          restored.memData = BigInt(state.memData);
        } else {
          restored[key] = state[key];
        }
      }
      
      // 确保所有必需的属性都存在
      const defaultState = {
        fetch: null,
        decode: null,
        execute: null,
        memory: null,
        writeback: null,
        aluA: 0,
        aluB: 0,
        alufun: 0,
        memAddr: 0,
        memData: 0,
        memRead: false,
        memWrite: false
      };
      
      return { ...defaultState, ...restored };
    },
    
    // 反汇编指定地址的指令
    disassembleAt(pc) {
      const cacheKey = pc;
      if (this.instructionCache.has(cacheKey)) {
        return this.instructionCache.get(cacheKey);
      }
      
      if (pc >= this.cpuState.mem.length) return null;
      
      const byte0 = this.readMemByte(pc);
      const icode = (byte0 >> 4) & 0xF;
      const ifun = byte0 & 0xF;
      
      if (icode > 15) return null;
      
      const needRegids = [2, 3, 4, 5, 6, 10, 11, 12].includes(icode);
      const needValC = [3, 4, 5, 7, 8].includes(icode);
      
      let length = 1;
      if (needRegids) length++;
      if (needValC) length += 8;
      
      if (pc + length > this.cpuState.mem.length) return null;
      
      const bytes = [];
      for (let i = 0; i < length && pc + i < this.cpuState.mem.length; i++) {
        bytes.push(this.readMemByte(pc + i));
      }
      
      let disasm = '';
      let rA = 15, rB = 15;
      let valC = 0n;
      
      if (needRegids && bytes.length > 1) {
        const regbyte = bytes[1];
        rA = (regbyte >> 4) & 0xF;
        rB = regbyte & 0xF;
      }
      
      if (needValC && bytes.length >= (needRegids ? 10 : 9)) {
        let val = 0n;
        const start = needRegids ? 2 : 1;
        for (let i = 0; i < 8; i++) {
          val |= BigInt(bytes[start + i]) << BigInt(i * 8);
        }
        valC = val;
      }
      
      switch (icode) {
        case 0:
          disasm = 'halt';
          break;
        case 1:
          disasm = 'nop';
          break;
        case 2:
          const cmovNames = ['rrmovq', 'cmovle', 'cmovl', 'cmove', 'cmovne', 'cmovge', 'cmovg'];
          disasm = `${cmovNames[ifun] || 'cmovXX'} ${this.getRegName(rA)}, ${this.getRegName(rB)}`;
          break;
        case 3:
          disasm = `irmovq $0x${valC.toString(16)}, ${this.getRegName(rB)}`;
          break;
        case 4:
          disasm = `rmmovq ${this.getRegName(rA)}, 0x${valC.toString(16)}(${this.getRegName(rB)})`;
          break;
        case 5:
          disasm = `mrmovq 0x${valC.toString(16)}(${this.getRegName(rB)}), ${this.getRegName(rA)}`;
          break;
        case 6:
          const opNames = ['addq', 'subq', 'andq', 'xorq'];
          disasm = `${opNames[ifun] || 'opq'} ${this.getRegName(rA)}, ${this.getRegName(rB)}`;
          break;
        case 7:
          const jmpNames = ['jmp', 'jle', 'jl', 'je', 'jne', 'jge', 'jg'];
          disasm = `${jmpNames[ifun] || 'jXX'} 0x${valC.toString(16)}`;
          break;
        case 8:
          disasm = `call 0x${valC.toString(16)}`;
          break;
        case 9:
          disasm = 'ret';
          break;
        case 10:
          disasm = `pushq ${this.getRegName(rA)}`;
          break;
        case 11:
          disasm = `popq ${this.getRegName(rA)}`;
          break;
        case 12:
          disasm = `prt ${this.getRegName(rA)}`;
          break;
        default:
          disasm = `未知指令 (icode=${icode})`;
      }
      
      const result = { bytes, length, disasm };
      this.instructionCache.set(cacheKey, result);
      return result;
    },
    
    // 获取机器码字符串
    getMachineCodeString(pc, length) {
      const bytes = [];
      for (let i = 0; i < length && pc + i < this.cpuState.mem.length; i++) {
        bytes.push(this.formatHex(this.readMemByte(pc + i), 2));
      }
      return bytes.join(' ');
    },
    
    // 生成当前指令的反汇编
    generateDisassembly() {
      if (!this.currentInstruction) return;
      
      const instr = this.currentInstruction;
      this.currentDisassembly = this.disassembleAt(instr.pc)?.disasm || '';
    },
    
    // 辅助方法
    readMemByte(addr) {
      if (addr >= 0 && addr < this.cpuState.mem.length) {
        return this.cpuState.mem[addr];
      }
      return 0;
    },
    
    readMemWord(addr) {
      let word = 0n;
      for (let i = 0; i < 8; i++) {
        if (addr + i < this.cpuState.mem.length) {
          word |= BigInt(this.cpuState.mem[addr + i]) << BigInt(i * 8);
        }
      }
      return word;
    },
    
    writeMemWord(addr, value) {
      for (let i = 0; i < 8; i++) {
        if (addr + i < this.cpuState.mem.length) {
          this.cpuState.mem[addr + i] = Number((value >> BigInt(i * 8)) & 0xFFn);
        }
      }
    },
    
    readReg(regId) {
      if (regId >= 0 && regId < 15) {
        return this.cpuState.regs[regId];
      }
      return 0n;
    },
    
    writeReg(regId, value) {
      if (regId >= 0 && regId < 15) {
        this.cpuState.regs[regId] = value;
      }
    },
    
    computeCondition(ifun) {
      const cc = this.cpuState.cc;
      
      switch (ifun) {
        case 0: return true;
        case 1: return (cc.SF ^ cc.OF) || cc.ZF;
        case 2: return (cc.SF ^ cc.OF);
        case 3: return cc.ZF;
        case 4: return !cc.ZF;
        case 5: return !(cc.SF ^ cc.OF);
        case 6: return !(cc.SF ^ cc.OF) && !cc.ZF;
        default: return false;
      }
    },
    
    getInstructionName(icode, ifun) {
      if (icode === 6) {
        const opNames = ['addq', 'subq', 'andq', 'xorq'];
        return opNames[ifun] || 'opq';
      } else if (icode === 7) {
        const jmpNames = ['jmp', 'jle', 'jl', 'je', 'jne', 'jge', 'jg'];
        return jmpNames[ifun] || 'jXX';
      } else if (icode === 2 && ifun !== 0) {
        const cmovNames = ['rrmovq', 'cmovle', 'cmovl', 'cmove', 'cmovne', 'cmovge', 'cmovg'];
        return cmovNames[ifun] || 'cmovXX';
      }
      return this.icodeNames[icode] || `icode_${icode}`;
    },
    
    getRegName(regId) {
      return regId < 16 ? this.regNames[regId] : `R${regId}`;
    },
    
    getICodeName(icode) {
      return icode < 16 ? this.icodeNames[icode] : `icode_${icode}`;
    },
    
    getALUFunName(alufun) {
      return alufun < 4 ? this.aluFunNames[alufun] : `alu_${alufun}`;
    },
    
    getStatName(stat) {
      return stat < 4 ? this.statNames[stat] : `stat_${stat}`;
    },
    
    getStatClass(stat) {
      const classes = ['stat-saok', 'stat-hlt', 'stat-adr', 'stat-ins'];
      return classes[stat] || 'stat-unknown';
    },
    
    getPhaseName(phase) {
      return phase >= 0 && phase < 6 ? this.phaseNames[phase] : '就绪';
    }
  }
};
</script>

<style scoped>
.y86-simulator {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  max-width: 100%;
  margin: 0;
  padding: 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
  color: #333;
  box-sizing: border-box;
}

h1 {
  text-align: center;
  color: white;
  margin-bottom: 30px;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
}

/* 文件上传区域 - 优化布局 */
.file-section {
  background: white;
  padding: 20px;
  border-radius: 10px;
  margin-bottom: 20px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
  align-items: center;
  box-sizing: border-box;
}

.file-upload-container {
  flex: 1;
  min-width: 200px;
}

.file-input {
  display: none;
}

.upload-btn {
  padding: 12px 24px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s;
  background: #4CAF50;
  color: white;
  width: 100%;
  font-size: 16px;
}

.upload-btn:hover {
  background: #45a049;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(76, 175, 80, 0.3);
}

.reset-btn {
  padding: 12px 24px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s;
  background: #f5f5f5;
  color: #333;
  border: 1px solid #ddd;
  min-width: 80px;
}

.reset-btn:hover {
  background: #e0e0e0;
}

.error-message {
  color: #f44336;
  font-weight: bold;
  flex-basis: 100%;
  margin-top: 10px;
}

/* 进度条区域 - 修复圆圈滑块 */
.progress-section {
  background: white;
  padding: 20px;
  border-radius: 10px;
  margin-bottom: 20px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  box-sizing: border-box;
}

.progress-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.progress-header label {
  font-weight: bold;
  color: #555;
  font-size: 14px;
}

.progress-status {
  font-size: 12px;
  font-weight: bold;
  padding: 4px 12px;
  border-radius: 12px;
  background: #e3f2fd;
  color: #1976d2;
}

.progress-status.completed {
  background: #e8f5e9;
  color: #2e7d32;
}

.progress-bar-container {
  position: relative;
  height: 24px;
  border-radius: 12px;
  overflow: hidden;
  background: #f5f5f5;
}

.progress-background {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: #f5f5f5;
  border-radius: 12px;
}

.progress-fill {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background: linear-gradient(90deg, #667eea 0%, #764ba2 100%);
  border-radius: 12px;
  transition: width 0.3s ease;
  z-index: 1;
}

.progress-bar-container.completed .progress-fill {
  background: linear-gradient(90deg, #4CAF50 0%, #2e7d32 100%);
}

.progress-thumb {
  position: absolute;
  top: 50%;
  transform: translate(-50%, -50%);
  z-index: 3;
  transition: left 0.3s ease;
}

.thumb-circle {
  width: 32px;
  height: 32px;
  background: white;
  border-radius: 50%;
  border: 3px solid #667eea;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.2);
  transition: all 0.3s ease;
}

.progress-bar-container.completed .thumb-circle {
  border-color: #4CAF50;
}

.thumb-value {
  position: absolute;
  top: -30px;
  left: 50%;
  transform: translateX(-50%);
  background: #667eea;
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: bold;
  white-space: nowrap;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  opacity: 0;
  transition: opacity 0.3s ease;
}

.progress-bar-container:hover .thumb-value {
  opacity: 1;
}

.progress-bar-container.completed .thumb-value {
  background: #4CAF50;
}

.progress-bar {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  opacity: 0;
  z-index: 4;
  cursor: pointer;
  margin: 0;
}

/* 自定义进度条滑块样式 */
.progress-bar::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: #667eea;
  cursor: pointer;
  border: 3px solid white;
  box-shadow: 0 2px 6px rgba(0,0,0,0.3);
  transition: all 0.3s ease;
}

.progress-bar.completed::-webkit-slider-thumb {
  background: #4CAF50;
}

.progress-bar::-moz-range-thumb {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: #667eea;
  cursor: pointer;
  border: 3px solid white;
  box-shadow: 0 2px 6px rgba(0,0,0,0.3);
}

.progress-bar.completed::-moz-range-thumb {
  background: #4CAF50;
}

.progress-bar::-webkit-slider-thumb:hover {
  transform: scale(1.1);
  box-shadow: 0 4px 8px rgba(0,0,0,0.3);
}

/* 执行状态 */
.execution-status {
  background: white;
  padding: 15px;
  border-radius: 10px;
  margin-bottom: 20px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  box-sizing: border-box;
}

.running-status {
  color: #2196F3;
  font-weight: bold;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
}

.finished-status {
  color: #4CAF50;
  font-weight: bold;
  text-align: center;
}

.stop-btn {
  padding: 5px 15px;
  background: #f44336;
  color: white;
  border: none;
  border-radius: 3px;
  cursor: pointer;
  font-weight: bold;
}

.stop-btn:hover {
  background: #da190b;
}

/* 控制按钮 */
.control-buttons {
  background: white;
  padding: 20px;
  border-radius: 10px;
  margin-bottom: 20px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  display: flex;
  gap: 15px;
  justify-content: center;
  flex-wrap: wrap;
  box-sizing: border-box;
}

.btn {
  padding: 12px 24px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  font-size: 16px;
  transition: all 0.3s;
  min-width: 150px;
  flex: 1;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.run-btn {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.run-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(102, 126, 234, 0.4);
}

.next-instruction-btn {
  background: #2196F3;
  color: white;
}

.next-instruction-btn:hover:not(:disabled) {
  background: #1976D2;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(33, 150, 243, 0.3);
}

.next-phase-btn {
  background: #FF9800;
  color: white;
}

.next-phase-btn:hover:not(:disabled) {
  background: #F57C00;
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(255, 152, 0, 0.3);
}

/* 主显示区域 */
.main-display {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px;
  margin-bottom: 30px;
  box-sizing: border-box;
}

@media (max-width: 1200px) {
  .main-display {
    grid-template-columns: 1fr;
  }
}

/* 通用区域样式 */
.section {
  background: white;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  box-sizing: border-box;
  overflow: auto;
  min-height: 600px;
  display: flex;
  flex-direction: column;
}

.section h2 {
  color: #667eea;
  margin-top: 0;
  border-bottom: 2px solid #667eea;
  padding-bottom: 10px;
  margin-bottom: 20px;
  flex-shrink: 0;
}

.section h3 {
  color: #555;
  margin-bottom: 15px;
  font-size: 16px;
}

/* 流水线阶段 */
.pipeline-stages {
  display: flex;
  flex-direction: column;
  gap: 15px;
  overflow-y: auto;
  flex: 1;
}

.pipeline-stage {
  background: #f8f9fa;
  border-radius: 8px;
  padding: 15px;
  border-left: 5px solid #ccc;
  transition: all 0.3s;
  flex-shrink: 0;
}

.pipeline-stage.active {
  background: #e3f2fd;
  border-left-color: #2196F3;
  box-shadow: 0 2px 8px rgba(33, 150, 243, 0.2);
}

.pipeline-stage h3 {
  margin-top: 0;
  color: #333;
  font-size: 14px;
}

.stage-content {
  font-size: 13px;
  line-height: 1.4;
}

.stage-content p {
  margin: 6px 0;
  font-size: 12px;
}

.stage-content strong {
  color: #667eea;
}

/* 指令信息 - 只保留内存视图 */
.instruction-section {
  display: flex;
  flex-direction: column;
}

.instruction-section .memory-view {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.instruction-section .memory-table {
  flex: 1;
}

/* 内存视图 */
.memory-view {
  background: #fff3e0;
  padding: 15px;
  border-radius: 8px;
  overflow: auto;
  display: flex;
  flex-direction: column;
  flex: 1;
}

.memory-table {
  border: 1px solid #ddd;
  border-radius: 5px;
  overflow: hidden;
  min-width: 600px;
  flex: 1;
  display: flex;
  flex-direction: column;
}

.memory-row {
  display: grid;
  grid-template-columns: 120px 1fr 200px;
  border-bottom: 1px solid #eee;
  padding: 8px;
  font-family: 'Courier New', monospace;
  font-size: 13px;
  align-items: center;
  flex-shrink: 0;
}

.memory-row.header {
  background: #f5f5f5;
  font-weight: bold;
  flex-shrink: 0;
}

.memory-row.current-pc {
  background: #e3f2fd;
  font-weight: bold;
}

.mem-addr {
  font-weight: bold;
  color: #2196F3;
  font-size: 12px;
}

.mem-bytes {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
}

.byte {
  padding: 2px 4px;
  border-radius: 3px;
  background: #f0f0f0;
  display: inline-block;
  font-size: 12px;
}

.mem-asm {
  color: #2e7d32;
  font-weight: bold;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 13px;
}

/* CPU状态 */
.cpu-state-section {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.registers {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 8px;
}

.register-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 10px;
}

.register-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background: white;
  border-radius: 4px;
  font-family: 'Courier New', monospace;
  font-size: 14px;
  border: 1px solid #eee;
}

.reg-name {
  font-weight: bold;
  color: #667eea;
  font-size: 14px;
}

.reg-value {
  color: #333;
  font-weight: bold;
  font-size: 14px;
  letter-spacing: 0.5px;
}

.reg-changed {
  color: #4CAF50;
  font-weight: bold;
  animation: pulse 1s infinite;
  font-size: 14px;
}

@keyframes pulse {
  0% { opacity: 1; }
  50% { opacity: 0.5; }
  100% { opacity: 1; }
}

.condition-codes {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 8px;
}

.cc-flags {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
}

.cc-flag {
  background: white;
  padding: 15px;
  border-radius: 5px;
  text-align: center;
  border: 2px solid #ddd;
}

.cc-flag.active {
  background: #e8f5e9;
  border-color: #4CAF50;
}

.cc-flag span {
  display: block;
}

.cc-flag span:first-child {
  font-weight: bold;
  margin-bottom: 5px;
  color: #555;
  font-size: 14px;
}

.cc-value {
  font-size: 20px;
  font-weight: bold;
  color: #333;
}

.status-section,
.program-counter {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 8px;
}

.status-display {
  padding: 15px;
  border-radius: 5px;
  text-align: center;
  font-weight: bold;
  font-size: 16px;
}

.stat-saok { 
  background: #e8f5e9; 
  color: #2e7d32; 
}
.stat-hlt { 
  background: #fff3e0; 
  color: #f57c00; 
}
.stat-adr { 
  background: #ffebee; 
  color: #c62828; 
}
.stat-ins { 
  background: #f3e5f5; 
  color: #7b1fa2; 
}

.pc-value {
  font-family: 'Courier New', monospace;
  font-size: 16px;
  background: white;
  padding: 15px;
  border-radius: 5px;
  text-align: center;
  border: 2px solid #667eea;
  color: #667eea;
  font-weight: bold;
}
</style>