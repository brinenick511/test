总体任务是使用Verilog语言设计RISC-V流水线CPU，支持RV32I中的大部分命令，带有数据冒险和控制冒险的处理单元。
已经大致写好了一个框架，有许多v文件，包括：“alu, controller, datapath, defines, mem, parts, pipelined, regfile, tb”。
这些v文件将在后面给出。
请你告诉我哪些文件需要改哪些部分。

以下是”defines.v“文件的内容：
```v
//=====================================================================
// Macro definitions
// ====================================================================

`define DEBUG

// ISA related

`define ADDR_SIZE			32 // the width of an address
`define INSTR_SIZE  32 // the length of an instruction
`define INSTR_WIDTH 5  // the bit width of an instruction

`define XLEN				    32 // the data width of a register
`define XLEN_WIDTH		5
`define RFREG_NUM			32 // the number of registers
`define RFIDX_WIDTH 5  // the width of a register index

`define IMEM_SIZE			1024
`define IMEM_SIZE_WIDTH		10
`define DMEM_SIZE			1024
`define DMEM_SIZE_WIDTH		10

//RV32I
`define OP_LUI		7'b0110111
`define OP_AUIPC	7'b0010111
`define OP_JAL		7'b1101111
`define OP_JALR		7'b1100111
`define OP_BRANCH	7'b1100011
`define OP_LOAD		7'b0000011
`define OP_STORE	7'b0100011
`define OP_ADDI		7'b0010011
`define OP_ADD		7'b0110011


`define FUNCT3_BEQ	3'b000
`define FUNCT3_BNE	3'b001
`define FUNCT3_BLT	3'b100
`define FUNCT3_BGE	3'b101
`define FUNCT3_BLTU	3'b110
`define FUNCT3_BGEU	3'b111

`define FUNCT3_LB	3'b000
`define FUNCT3_LH	3'b001
`define FUNCT3_LW	3'b010
`define FUNCT3_LBU	3'b100
`define FUNCT3_LHU	3'b101

`define FUNCT3_SB	3'b000
`define FUNCT3_SH	3'b001
`define FUNCT3_SW	3'b010

`define FUNCT3_ADDI	3'b000
`define FUNCT3_SLTI	3'b010
`define FUNCT3_SLTIU	3'b011
`define FUNCT3_XORI	3'b100
`define FUNCT3_ORI	3'b110
`define FUNCT3_ANDI	3'b111

`define FUNCT3_SL	3'b001
`define FUNCT3_SR	3'b101

`define FUNCT7_SLLI	7'b0000000
`define FUNCT7_SRLI	7'b0000000
`define FUNCT7_SRAI	7'b0100000

`define FUNCT3_ADD	3'b000
`define FUNCT3_SLL	3'b001
`define FUNCT3_SLT	3'b010
`define FUNCT3_SLTU	3'b011
`define FUNCT3_XOR	3'b100
`define FUNCT3_SR	3'b101
`define FUNCT3_OR	3'b110
`define FUNCT3_AND	3'b111

`define FUNCT7_ADD	7'b0000000
`define FUNCT7_SUB	7'b0100000

`define FUNCT7_SRL	7'b0000000
`define FUNCT7_SRA	7'b0100000

//RV64I
`define FUNCT3_LWU	3'b110
`define FUNCT3_LD	3'b011

`define FUNCT3_SD	3'b011

`define OP_ADDIW	7'b0011011
`define OP_ADDW		7'b0111011

//ID ALU bus
`define ID_ALU_WIDTH	16

//ALU CTRL
`define	ALU_CTRL_MOVEA	4'b0000

`define	ALU_CTRL_ADD	4'b0001
`define	ALU_CTRL_ADDU	4'b0010

`define	ALU_CTRL_OR		4'b0011
`define	ALU_CTRL_XOR	4'b0100
`define	ALU_CTRL_AND	4'b0101

`define	ALU_CTRL_SLL	4'b0110
`define	ALU_CTRL_SRL	4'b0111
`define	ALU_CTRL_SRA	4'b1000

`define	ALU_CTRL_SUB	4'b1001
`define	ALU_CTRL_SUBU	4'b1010
`define	ALU_CTRL_SLT	4'b1011
`define	ALU_CTRL_SLTU	4'b1100

`define	ALU_CTRL_LUI	4'b1101
`define	ALU_CTRL_AUIPC	4'b1110

`define	ALU_CTRL_ZERO	4'b1111

//IMM CTRL itype, stype, btype, utype, jtype
`define IMM_CTRL_ITYPE	5'b10000
`define IMM_CTRL_STYPE	5'b01000
`define IMM_CTRL_BTYPE	5'b00100
`define IMM_CTRL_UTYPE	5'b00010
`define IMM_CTRL_JTYPE	5'b00001
```
以下是”alu.v“文件的内容：
```v
//=====================================================================
// The alu module implements the core's ALU.
// ====================================================================

`include "xgriscv_defines.v"
module alu(
	input	[`XLEN-1:0]	a, b, 
	input	[4:0]  		shamt,		//???
	input	[3:0]   	aluctrl, 

	output reg [`XLEN-1:0]	aluout,
	output       	overflow,
	output 			zero,
	output 			lt,
	output 			ge
	);

	wire op_unsigned = ~aluctrl[3]&~aluctrl[2]&aluctrl[1]&~aluctrl[0]	//ALU_CTRL_ADDU	4'b0010
					| aluctrl[3]&~aluctrl[2]&aluctrl[1]&~aluctrl[0] 	//ALU_CTRL_SUBU	4'b1010
					| aluctrl[3]&aluctrl[2]&~aluctrl[1]&~aluctrl[0]; 	//ALU_CTRL_SLTU	4'b1100

	wire [`XLEN-1:0] 	b2;
	wire [`XLEN:0] 		sum; //adder of length XLEN+1
  
	assign b2 = aluctrl[3] ? ~b:b; 
	assign sum = (op_unsigned & ({1'b0, a} + {1'b0, b2} + aluctrl[3]))
				| (~op_unsigned & ({a[`XLEN-1], a} + {b2[`XLEN-1], b2} + aluctrl[3]));
				// aluctrl[3]=0 if add, or 1 if sub, don't care if other

	always@(*)
		case(aluctrl[3:0])
		`ALU_CTRL_MOVEA: 	aluout <= a;
		`ALU_CTRL_ADD: 		aluout <= sum[`XLEN-1:0];
		`ALU_CTRL_ADDU:		aluout <= sum[`XLEN-1:0];
		
		`ALU_CTRL_LUI:		aluout <= sum[`XLEN-1:0]; //a = 0, b = immout
		`ALU_CTRL_AUIPC:	aluout <= sum[`XLEN-1:0]; //a = pc, b = immout

		//`ALU_CTRL_ZERO		
		default: 			aluout <= `XLEN'b0; 
	 endcase
	    
	assign overflow = sum[`XLEN-1] ^ sum[`XLEN];
	assign zero = (aluout == `XLEN'b0);
	assign lt = aluout[`XLEN-1];
	assign ge = ~aluout[`XLEN-1];
endmodule
```
以下是”controller.v“文件的内容：
```v
//=====================================================================
// The controller module generates the controlling signals.
// ====================================================================

`include "xgriscv_defines.v"

module controller(
  input                     clk, reset,
  input [6:0]	              opcode,
  input [2:0]               funct3,
  input [6:0]               funct7,
  input [`RFIDX_WIDTH-1:0]  rd, rs1,
  input [11:0]              imm,  
  input                     zero, lt,              // from cmp in the decode stage

  output [4:0]              immctrl,               // for the ID stage
  output                    itype, jal, jalr, bunsigned, pcsrc,
  output reg  [3:0]         aluctrl,               // for the EX stage 
  output [1:0]              alusrca,
  output                    alusrcb,
  output                    memwrite, lunsigned,   // for the MEM stage
  output [1:0]              lwhb, swhb,
  output                    memtoreg, regwrite     // for the WB stage
  );

  wire rv32_lui		= (opcode == `OP_LUI);
  wire rv32_auipc	= (opcode == `OP_AUIPC);
  wire rv32_jal		= (opcode == `OP_JAL);
  wire rv32_jalr	= (opcode == `OP_JALR);
  wire rv32_branch= (opcode == `OP_BRANCH);
  wire rv32_load	= (opcode == `OP_LOAD); 
  wire rv32_store	= (opcode == `OP_STORE);
  wire rv32_addri	= (opcode == `OP_ADDI);
  wire rv32_addrr = (opcode == `OP_ADD);

  wire rv32_beq		= ((opcode == `OP_BRANCH) & (funct3 == `FUNCT3_BEQ));
  wire rv32_bne		= ((opcode == `OP_BRANCH) & (funct3 == `FUNCT3_BNE));
  wire rv32_blt		= ((opcode == `OP_BRANCH) & (funct3 == `FUNCT3_BLT));
  wire rv32_bge		= ((opcode == `OP_BRANCH) & (funct3 == `FUNCT3_BGE));
  wire rv32_bltu	= ((opcode == `OP_BRANCH) & (funct3 == `FUNCT3_BLTU));
  wire rv32_bgeu	= ((opcode == `OP_BRANCH) & (funct3 == `FUNCT3_BGEU));

  wire rv32_lb		= ((opcode == `OP_LOAD) & (funct3 == `FUNCT3_LB));
  wire rv32_lh		= ((opcode == `OP_LOAD) & (funct3 == `FUNCT3_LH));
  wire rv32_lw		= ((opcode == `OP_LOAD) & (funct3 == `FUNCT3_LW));
  wire rv32_lbu		= ((opcode == `OP_LOAD) & (funct3 == `FUNCT3_LBU));
  wire rv32_lhu		= ((opcode == `OP_LOAD) & (funct3 == `FUNCT3_LHU));

  wire rv32_sb		= ((opcode == `OP_STORE) & (funct3 == `FUNCT3_SB));
  wire rv32_sh		= ((opcode == `OP_STORE) & (funct3 == `FUNCT3_SH));
  wire rv32_sw		= ((opcode == `OP_STORE) & (funct3 == `FUNCT3_SW));

  wire rv32_addi	= ((opcode == `OP_ADDI) & (funct3 == `FUNCT3_ADDI));
  wire rv32_slti	= ((opcode == `OP_ADDI) & (funct3 == `FUNCT3_SLTI));
  wire rv32_sltiu	= ((opcode == `OP_ADDI) & (funct3 == `FUNCT3_SLTIU));
  wire rv32_xori	= ((opcode == `OP_ADDI) & (funct3 == `FUNCT3_XORI));
  wire rv32_ori	  = ((opcode == `OP_ADDI) & (funct3 == `FUNCT3_ORI));
  wire rv32_andi	= ((opcode == `OP_ADDI) & (funct3 == `FUNCT3_ANDI));
  wire rv32_slli	= ((opcode == `OP_ADDI) & (funct3 == `FUNCT3_SL) & (funct7 == `FUNCT7_SLLI));
  wire rv32_srli	= ((opcode == `OP_ADDI) & (funct3 == `FUNCT3_SR) & (funct7 == `FUNCT7_SRLI));
  wire rv32_srai	= ((opcode == `OP_ADDI) & (funct3 == `FUNCT3_SR) & (funct7 == `FUNCT7_SRAI));

  wire rv32_add		= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_ADD) & (funct7 == `FUNCT7_ADD));
  wire rv32_sub		= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_ADD) & (funct7 == `FUNCT7_SUB));
  wire rv32_sll		= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_SLL));
  wire rv32_slt		= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_SLT));
  wire rv32_sltu	= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_SLTU));
  wire rv32_xor		= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_XOR));
  wire rv32_srl		= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_SR) & (funct7 == `FUNCT7_SRL));
  wire rv32_sra		= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_SR) & (funct7 == `FUNCT7_SRA));
  wire rv32_or		= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_OR));
  wire rv32_and		= ((opcode == `OP_ADD) & (funct3 == `FUNCT3_AND));

  wire rv32_rs1_x0= (rs1 == 5'b00000);
  wire rv32_rd_x0 = (rd  == 5'b00000);
  wire rv32_nop		= rv32_addi & rv32_rs1_x0 & rv32_rd_x0 & (imm == 12'b0); //addi x0, x0, 0 is nop

  assign itype = rv32_addi;

  wire stype = 0;

  wire btype = 0;

  wire utype = rv32_lui | rv32_auipc;

  wire jtype = 0;

  assign immctrl = {itype, stype, btype, utype, jtype};

  assign jal = 0;
  
  assign jalr = 0;

  assign bunsigned = 0;

  assign pcsrc = 0;

  assign alusrca = rv32_lui ? 2'b01 : (rv32_auipc ? 2'b10 : 2'b00);

  assign alusrcb = rv32_lui || rv32_auipc || rv32_addi;

  assign memwrite = 0;

  assign swhb = 0;

  assign lwhb = 0;

  assign lunsigned = 0;

  assign memtoreg = 0;

  assign regwrite = rv32_lui | rv32_auipc | rv32_addi;


  always @(*)
    case(opcode)
      `OP_LUI:    aluctrl <= `ALU_CTRL_ADD;
      `OP_AUIPC:  aluctrl <= `ALU_CTRL_ADD;
      `OP_ADDI:	  case(funct3)
                    `FUNCT3_ADDI:	aluctrl <= `ALU_CTRL_ADD;
                    default:		aluctrl <= `ALU_CTRL_ZERO;	
                  endcase
      default:  aluctrl <= `ALU_CTRL_ZERO;
	 endcase

endmodule
```
以下是”datapath.v“文件的内容：
```v
//=====================================================================
// The datapath of the pipeline.
// ====================================================================

`include "xgriscv_defines.v"

module datapath(
	input                    clk, reset,

	input [`INSTR_SIZE-1:0]  instrF, 	 // from instructon memory
	output[`ADDR_SIZE-1:0] 	 pcF, 		   // to instruction memory

	input [`XLEN-1:0]	       readdataM, // from data memory: read data
  output[`XLEN-1:0]        aluoutM, 	 // to data memory: address
 	output[`XLEN-1:0]	       writedataM,// to data memory: write data
  output			                memwriteM,	// to data memory: write enable
 	output [`ADDR_SIZE-1:0]  pcM,       // to data memory: pc of the write instruction
 	
 	output [`ADDR_SIZE-1:0]  pcW,       // to testbench
  
	
	// from controller
	input [4:0]		            immctrlD,
	input			                 itype, jalD, jalrD, bunsignedD, pcsrcD,
	input [3:0]		            aluctrlD,
	input [1:0]		            alusrcaD,
	input			                 alusrcbD,
	input			                 memwriteD, lunsignedD,
	input [1:0]		          	 lwhbD, swhbD,  
	input          		        memtoregD, regwriteD,
	
  	// to controller
	output [6:0]		           opD,
	output [2:0]		           funct3D,
	output [6:0]		           funct7D,
	output [4:0] 		          rdD, rs1D,
	output [11:0]  		        immD,
	output 	       		        zeroD, ltD
	);

	// next PC logic (operates in fetch and decode)
	wire [`ADDR_SIZE-1:0]	 pcplus4F, nextpcF, pcbranchD, pcadder2aD, pcadder2bD, pcbranch0D;
	mux2 #(`ADDR_SIZE)	    pcsrcmux(pcplus4F, pcbranchD, pcsrcD, nextpcF);
	
	// Fetch stage logic
	pcenr      	 pcreg(clk, reset, 1'b1, nextpcF, pcF);
	addr_adder  	pcadder1(pcF, `ADDR_SIZE'b100, pcplus4F);

	///////////////////////////////////////////////////////////////////////////////////
	// IF/ID pipeline registers
	wire [`INSTR_SIZE-1:0]	instrD;
	wire [`ADDR_SIZE-1:0]	pcD, pcplus4D;
	wire flushD = 0;

	floprc #(`INSTR_SIZE) 	pr1D(clk, reset, flushD, instrF, instrD);     // instruction
	floprc #(`ADDR_SIZE)	  pr2D(clk, reset, flushD, pcF, pcD);           // pc
	floprc #(`ADDR_SIZE)	  pr3D(clk, reset, flushD, pcplus4F, pcplus4D); // pc+4

	// Decode stage logic
	wire [`RFIDX_WIDTH-1:0] rs2D;
	assign  opD 	= instrD[6:0];
	assign  rdD     = instrD[11:7];
	assign  funct3D = instrD[14:12];
	assign  rs1D    = instrD[19:15];
	assign  rs2D   	= instrD[24:20];
	assign  funct7D = instrD[31:25];
	assign  immD    = instrD[31:20];

	// immediate generate
	wire [11:0]  iimmD = instrD[31:20];
	wire [11:0]		simmD	= 12'b0;
	wire [11:0]  bimmD	= 20'b0;
	wire [19:0]		uimmD	= instrD[31:12];
	wire [19:0]  jimmD	= 20'b0;
	wire [`XLEN-1:0]	immoutD, shftimmD;
	wire [`XLEN-1:0]	rdata1D, rdata2D, wdataW;
	wire [`RFIDX_WIDTH-1:0]	waddrW;

	imm 	im(iimmD, simmD, bimmD, uimmD, jimmD, immctrlD, immoutD);

	// register file (operates in decode and writeback)
	regfile rf(clk, rs1D, rs2D, rdata1D, rdata2D, regwriteW, waddrW, wdataW, pcW);

	///////////////////////////////////////////////////////////////////////////////////
	// ID/EX pipeline registers

	// for control signals
	wire       regwriteE, memwriteE, alusrcbE;
	wire [1:0] alusrcaE;
	wire [3:0] aluctrlE;
	wire 	     flushE = 0;
	floprc #(9) regE(clk, reset, flushE,
                  {regwriteD, memwriteD, alusrcaD, alusrcbD, aluctrlD}, 
                  {regwriteE, memwriteE, alusrcaE, alusrcbE, aluctrlE});
  
	// for data
	wire [`XLEN-1:0]	srca1E, srcb1E, immoutE, srcaE, srcbE, aluoutE;
	wire [`RFIDX_WIDTH-1:0] rdE;
	wire [`ADDR_SIZE-1:0] 	pcE, pcplus4E;
	floprc #(`XLEN) 	pr1E(clk, reset, flushE, rdata1D, srca1E);        	// data from rs1
	floprc #(`XLEN) 	pr2E(clk, reset, flushE, rdata2D, srcb1E);         // data from rs2
	floprc #(`XLEN) 	pr3E(clk, reset, flushE, immoutD, immoutE);        // imm output
 	floprc #(`RFIDX_WIDTH)  pr6E(clk, reset, flushE, rdD, rdE);         // rd
 	floprc #(`ADDR_SIZE)	pr8E(clk, reset, flushE, pcD, pcE);            // pc
 	floprc #(`ADDR_SIZE)	pr9E(clk, reset, flushE, pcplus4D, pcplus4E);  // pc+4

	// execute stage logic
	mux3 #(`XLEN)  srcamux(srca1E, 0, pcE, alusrcaE, srcaE);     // alu src a mux
	mux2 #(`XLEN)  srcbmux(srcb1E, immoutE, alusrcbE, srcbE);			 // alu src b mux

	alu alu(srcaE, srcbE, 5'b0, aluctrlE, aluoutE, overflowE, zeroE, ltE, geE);

	///////////////////////////////////////////////////////////////////////////////////
	// EX/MEM pipeline registers
	// for control signals
	wire 		regwriteM;
	wire 		flushM = 0;
	floprc #(2) 	regM(clk, reset, flushM,
                  	{regwriteE, memwriteE},
                  	{regwriteM, memwriteM});

	// for data
 	wire [`RFIDX_WIDTH-1:0]	 rdM;
	floprc #(`XLEN) 	        pr1M(clk, reset, flushM, aluoutE, aluoutM);
	floprc #(`RFIDX_WIDTH) 	 pr2M(clk, reset, flushM, rdE, rdM);
	floprc #(`ADDR_SIZE)	    pr3M(clk, reset, flushM, pcE, pcM);            // pc
	
	// mem stage logic
	

  ///////////////////////////////////////////////////////////////////////////////////
  // MEM/WB pipeline registers
  // for control signals
  wire flushW = 0;
	floprc #(1) regW(clk, reset, flushW, {regwriteM}, {regwriteW});

  // for data
  wire[`XLEN-1:0]		       aluoutW;
  wire[`RFIDX_WIDTH-1:0]	 rdW;

  floprc #(`XLEN) 	       pr1W(clk, reset, flushW, aluoutM, aluoutW);
  floprc #(`RFIDX_WIDTH)  pr2W(clk, reset, flushW, rdM, rdW);
  floprc #(`ADDR_SIZE)	   pr3W(clk, reset, flushW, pcM, pcW);            // pc
	
	// write-back stage logic
	assign wdataW = aluoutW;
	assign waddrW = rdW;

endmodule
```
以下是”mem.v“文件的内容：
```v
//=====================================================================
// The instruction memory and data memory.
// ====================================================================

`include "xgriscv_defines.v"

module imem(input  [`ADDR_SIZE-1:0]   a,
            output [`INSTR_SIZE-1:0]  rd);

  reg  [`INSTR_SIZE-1:0] RAM[`IMEM_SIZE-1:0];

  assign rd = RAM[a[11:2]]; // instruction size aligned
endmodule


module dmem(input           	         clk, we,
            input  [`XLEN-1:0]        a, wd,
            input  [`ADDR_SIZE-1:0] 	 pc,
            output [`XLEN-1:0]        rd);

  reg  [31:0] RAM[1023:0];

  assign rd = RAM[a[11:2]]; // word aligned

  always @(posedge clk)
    if (we)
      begin
        RAM[a[11:2]] <= wd;          	  // sw
        // DO NOT CHANGE THIS display LINE!!!
        // 不要修改下面这行display语句！！！
        // 对于所有的store指令，都输出位于写入目标地址四字节对齐处的32位数据，不需要修改下面的display语句
        /**********************************************************************/
        $display("pc = %h: dataaddr = %h, memdata = %h", pc, {a[31:2],2'b00}, RAM[a[11:2]]);
        /**********************************************************************/
  	  end
endmodule
```
以下是”parts.v“文件的内容：
```v
//=====================================================================
// The gadgets.
// ====================================================================

`include "xgriscv_defines.v"

// pc register with write enable
module pcenr (
	input             		clk, reset,
	input             		en,
	input      [`XLEN-1:0]	d, 
	output reg [`XLEN-1:0]	q);
 
	always @(posedge clk, posedge reset)
	// if      (reset) q <= 0;
    if (reset) 
    	q <= `ADDR_SIZE'h80000000 ;
    else if (en)    
    	q <=  d;
endmodule

// adder for address calculation
module addr_adder (
	input  [`ADDR_SIZE-1:0] a, b,
	output [`ADDR_SIZE-1:0] y);

	assign  y = a + b;
endmodule

// flop with reset and clear control
module floprc #(parameter WIDTH = 8)
              (input                  clk, reset, clear,
               input      [WIDTH-1:0] d, 
               output reg [WIDTH-1:0] q);

  always @(posedge clk, posedge reset)
    if (reset)      q <= 0;
    else if (clear) q <= 0;
    else            q <= d;
endmodule

// flop with reset, enable and clear control
module flopenrc #(parameter WIDTH = 8)
                 (input                  clk, reset,
                  input                  en, clear,
                  input      [WIDTH-1:0] d, 
                  output reg [WIDTH-1:0] q);
 
  always @(posedge clk, posedge reset)
    if      (reset) q <= 0;
    else if (clear) q <= 0;
    else if (en)    q <= d;
endmodule

// flop with reset and enable control
module flopenr #(parameter WIDTH = 8)
                (input                  clk, reset,
                 input                  en,
                 input      [WIDTH-1:0] d, 
                 output reg [WIDTH-1:0] q);
 
  always @(posedge clk, posedge reset)
    if      (reset) q <= 0;
    else if (en)    q <=  d;
endmodule

module mux2 #(parameter WIDTH = 8)
             (input  [WIDTH-1:0] d0, d1, 
              input              s, 
              output [WIDTH-1:0] y);

  assign y = s ? d1 : d0; 
endmodule

module mux3 #(parameter WIDTH = 8)
             (input  [WIDTH-1:0] d0, d1, d2,
              input  [1:0]       s, 
              output [WIDTH-1:0] y);

  assign  y = s[1] ? d2 : (s[0] ? d1 : d0); 
endmodule

module mux4 #(parameter WIDTH = 8)
             (input  [WIDTH-1:0] d0, d1, d2, d3,
              input  [1:0]       s, 
              output reg [WIDTH-1:0] y);

	always @( * )
	begin
      case(s)
         2'b00: y <= d0;
         2'b01: y <= d1;
         2'b10: y <= d2;
         2'b11: y <= d3;
      endcase
	end
endmodule

module mux5 #(parameter WIDTH = 8)
             (input		[WIDTH-1:0] d0, d1, d2, d3, d4,
              input		[2:0]       s, 
              output reg	[WIDTH-1:0] y);

	always @( * )
	begin
      case(s)
	    3'b000: y <= d0;
	    3'b001: y <= d1;
	    3'b010: y <= d2;
	    3'b011: y <= d3;
	    3'b100: y <= d4;
      endcase
//    $display("mux5 d0=%h, d1=%h, d2=%h, d3=%h, d4=%h, s=%b, y=%h", d0,d1,d2,d3,d4,s,y);
    end  
endmodule

module mux6 #(parameter WIDTH = 8)
           (input  [WIDTH-1:0] 	d0, d1, d2, d3, d4, d5,
            input  [2:0] 		s,
         	  output reg [WIDTH-1:0]	y);

	always@( * )
	begin
	  case(s)
		3'b000: y <= d0;
		3'b001: y <= d1;
		3'b010: y <= d2;
		3'b011: y <= d3;
		3'b100: y <= d4;
		3'b101: y <= d5;
	  endcase
	end
endmodule

module imm (
	input	[11:0]			iimm, //instr[31:20], 12 bits
	input	[11:0]			simm, //instr[31:25, 11:7], 12 bits
	input	[11:0]			bimm, //instrD[31], instrD[7], instrD[30:25], instrD[11:8], 12 bits
	input	[19:0]			uimm,
	input	[19:0]			jimm,
	input	[4:0]			 immctrl,

	output	reg [`XLEN-1:0] 	immout);
  always  @(*)
	 case (immctrl)
		`IMM_CTRL_ITYPE:	immout <= {{{`XLEN-12}{iimm[11]}}, iimm[11:0]};
		`IMM_CTRL_UTYPE:	immout <= {uimm[19:0], 12'b0};
		default:			      immout <= `XLEN'b0;
	 endcase
endmodule

// shift left by 1 for address calculation
module sl1(
	input  [`ADDR_SIZE-1:0] a,
	output [`ADDR_SIZE-1:0] y);

  assign  y = {a[`ADDR_SIZE-2:0], 1'b0};
endmodule

// comparator for branch
module cmp(
  input [`XLEN-1:0] a, b,
  input             op_unsigned,
  output            zero,
  output            lt);

  assign zero = (a == b);
  assign lt = (!op_unsigned & ($signed(a) < $signed(b))) | (op_unsigned & (a < b));
endmodule

module ampattern (input [1:0] addr, input [1:0] swhb, output reg [3:0] amp); //amp: access memory pattern
  always@(*)
  case (swhb)
    2'b01: amp <= 4'b1111;
    2'b10: if (addr[1]) amp <= 4'b1100;
           else         amp <= 4'b0011; //addr[0]
    2'b11: case (addr)
              2'b00: amp <= 4'b0001;
              2'b01: amp <= 4'b0010;
              2'b10: amp <= 4'b0100;
              2'b11: amp <= 4'b1000;
           endcase
    default: amp <= 4'b1111;// it shouldn't happen
  endcase
endmodule
```
以下是”pipelined.v“文件的内容：
```v
//=====================================================================
// The overall of the pipelined xg-riscv implementation.
// ====================================================================

`include "xgriscv_defines.v"
module xgriscv_pipeline(
  input                   clk, reset,
  output[`ADDR_SIZE-1:0]  pcW);
  
  wire [31:0]    instr;
  wire [31:0]    pcF, pcM;
  wire           memwrite;
  wire [3:0]     amp;
  wire [31:0]    addr, writedata, readdata;
   
  
  imem U_imem(pcF, instr);

  dmem U_dmem(clk, memwrite, addr, writedata, pcM, readdata);
  
  xgriscv U_xgriscv(clk, reset, pcF, instr, memwrite, amp, addr, writedata, pcM, pcW, readdata);
  
endmodule

// xgriscv: a pipelined riscv processor
module xgriscv(input         			        clk, reset,
               output [31:0] 			        pcF,
               input  [`INSTR_SIZE-1:0] instr,
               output					              memwrite,
               output [3:0]  			        amp,
               output [`ADDR_SIZE-1:0] 	daddr, 
               output [`XLEN-1:0] 		    writedata,
               output [`ADDR_SIZE-1:0] 	pcM,
               output [`ADDR_SIZE-1:0] 	pcW,
               input  [`XLEN-1:0] 		    readdata);
	
  wire [6:0]  opD;
 	wire [2:0]  funct3D;
	wire [6:0]  funct7D;
  wire [4:0]  rdD, rs1D;
  wire [11:0] immD;
  wire        zeroD, ltD;
  wire [4:0]  immctrlD;
  wire        itypeD, jalD, jalrD, bunsignedD, pcsrcD;
  wire [3:0]  aluctrlD;
  wire [1:0]  alusrcaD;
  wire        alusrcbD;
  wire        memwriteD, lunsignedD;
  wire [1:0]  swhbD, lwhbD;
  wire        memtoregD, regwriteD;

  controller  c(clk, reset, opD, funct3D, funct7D, rdD, rs1D, immD, zeroD, ltD,
              immctrlD, itypeD, jalD, jalrD, bunsignedD, pcsrcD, 
              aluctrlD, alusrcaD, alusrcbD, 
              memwriteD, lunsignedD, lwhbD, swhbD, 
              memtoregD, regwriteD);


  datapath    dp(clk, reset,
              instr, pcF,
              readdata, daddr, writedata, memwrite, pcM, pcW,
              immctrlD, itypeD, jalD, jalrD, bunsignedD, pcsrcD, 
              aluctrlD, alusrcaD, alusrcbD, 
              memwriteD, lunsignedD, lwhbD, swhbD, 
              memtoregD, regwriteD, 
              opD, funct3D, funct7D, rdD, rs1D, immD, zeroD, ltD);

endmodule

```
以下是”regfile.v“文件的内容：
```v
//=====================================================================
// The regfile module implements the core's general purpose registers file.
// ====================================================================

`include "xgriscv_defines.v"

module regfile(
  input                      	clk,
  input  [`RFIDX_WIDTH-1:0]  	ra1, ra2,
  output [`XLEN-1:0]          rd1, rd2,

  input                      	we3, 
  input  [`RFIDX_WIDTH-1:0]  	wa3,
  input  [`XLEN-1:0]          wd3,
  
  input  [`ADDR_SIZE-1:0] 	   pc
  );

  reg [`XLEN-1:0] rf[`RFREG_NUM-1:0];

  // three ported register file
  // read two ports combinationally
  // write third port on falling edge of clock
  // register 0 hardwired to 0

  always @(negedge clk)
    if (we3 && wa3!=0)
      begin
        rf[wa3] <= wd3;
        // DO NOT CHANGE THIS display LINE!!!
        // 不要修改下面这行display语句！！！
        /**********************************************************************/
        $display("pc = %h: x%d = %h", pc, wa3, wd3);
        /**********************************************************************/
      end

  assign rd1 = (ra1 != 0) ? rf[ra1] : 0;
  assign rd2 = (ra2 != 0) ? rf[ra2] : 0;
endmodule

```
以下是”tb.v“文件的内容：
```v
//=====================================================================
// testbench for simulation
// ====================================================================

`include "xgriscv_defines.v"

module xgriscv_tb();
    
   reg                  clk, rstn;
   wire[`ADDR_SIZE-1:0] pcW;
    
   // instantiation of xgriscv 
   xgriscv_pipeline xgriscvp(clk, rstn, pcW);

   integer counter = 0;
   
   initial begin
      $readmemh("riscv32_sim1.dat", xgriscvp.U_imem.RAM);
      clk = 1;
      rstn = 1;
      #5 ;
      rstn = 0;
   end
   
   always begin
      #(50) clk = ~clk;
     
      if (clk == 1'b1) 
      begin
         counter = counter + 1;
         // comment these four lines for online judge
         //$display("clock: %d", counter);
         //$display("pc:\t\t%h", xgriscvp.pcF);
         //$display("instr:\t%h", xgriscvp.instr);
         //$display("pcw:\t\t%h", pcW);
         if (pcW == 32'h80000078) // set to the address of the last instruction
          begin
            //$display("pcW:\t\t%h", pcW);
            //$finish;
            $stop;
          end
      end
      
   end //end always
   
endmodule

```
