Exercise 1

Bug 1:  add t1, s0, x0 //The first instruction under mapLoop section 
The address of the array of current node is save on the address hold by s0 so to access the address of the array, the instruction should be lw t1, 0(s0)

Bug 2: add t1,s0, t0 //The third instruction under mapLoop section
The offset of an address in riscv should be multiplied by 4 if the offset is indicated by index so the instruction should be slli t3, t0, 2 followed by add t1, t1, t3. 

Bug 3: 1a a0, 8(s0)  //The third instruction under mapLoop section
1a instruction should work with a label rather than an address. To fix this problem, use lw a0, 8(s0) instead.

Bug 4: lw a1, 0(s1) //The tenth instruction under mapLoop section 
The address of the function is saved by s1 rather than saved on the address held by s1 so the instruction should be addi a1, s1, 0.

Bug 5: mult t1, a0, a0 //The first instruction under mystery section
	add a0, t1, a0//The second instruction under mystery section
The value of t1 will not restore after mystery function executed so the mystery function will mess up the address of the array. Change t1 to any other temporary register that are not being used yet so change t1 to t2. 

Modified Code:

.data 
arrays:		.word	5, 6, 7, 8, 9 
		.word	1, 2, 3, 4, 7 
		.word	5, 2, 7, 4, 3 
		.word	1, 6, 3, 8, 4 
		.word	5, 2, 7, 8, 1 
		 
start_msg:	.asciiz "Lists before: \n" 
end_msg:	.asciiz "Lists after: \n" 
 
.text 
main:	jal	create_default_list 
	add	s0, a0, x0	# v0 = s0 is head of node list 
 
	#print "lists before: " 
	la	a1, start_msg 
	li	a0, 4 
	ecall 
 
	#print the list 
	add	a0, s0, x0 
	jal	print_list 
 
	# print a newline 
	jal	print_newline 
 
	#issue the map call 
	add	a0, s0, x0	# load the address of the first node into a0 
	la 	a1, mystery	# load the address of the function into a1 
 
	jal	map 
 
	# print "lists after: " 
	la	a1, end_msg 
	li	a0, 4 
	ecall 
 
	#print the list 
	add	a0, s0, x0 
	jal	print_list 
 
	li	a0, 10 
	ecall 
 
map: 
	addi sp, sp, -12 
	sw	ra, 0(sp) 
	sw	s1, 4(sp) 
	sw	s0, 8(sp) 
 
	beq	a0, x0, done	# if we were given a null pointer, we're done. 
 
	add	s0, a0, x0	# save address of this node in s0 
	add	s1, a1, x0	# save address of function in s1 
	add	t0, x0, x0	# t0 is a counter 
 
	# remember that each node is 12 bytes long: 4 for the array pointer, 4 for the size of the array, and 4 more for the pointer to the next node 
mapLoop: 
	lw	t1, 0(s0)		# load the address of the array of current node into t1 
	lw	t2, 4(s0)		# load the size of the node's array into t2 
    slli t3, t0, 2
	add	t1, t1, t3		# offset the array address by the count 
	lw	a0, 0(t1)		# load the value at that address into a0 
	 
	jalr	s1			# call the function on that value. 
	 
	sw	a0, 0(t1)		# store the returned value back into the array 
	addi	t0, t0, 1		# increment the count 
	bne	t0, t2, mapLoop	# repeat if we haven't reached the array size yet 
	 
	lw a0, 8(s0)		# load the address of the next node into a0 
	addi a1, s1, 0		# put the address of the function back into a1 to prepare for the recursion 
	jal 	map			# recurse 
 
done: 
	lw	s0, 8(sp) 
	lw	s1, 4(sp) 
	lw	ra, 0(sp) 
	addi	sp, sp, 12 
	jr	ra 
 
mystery: 
	mul	t4, a0, a0 
	add	a0, t4, a0 
	jr	ra 
 
create_default_list: 
	addi	sp, sp, -4 
	sw	ra, 0(sp) 
	li	s0, 0		# pointer to the last node we handled 
	li	s1, 0		# number of nodes handled 
	li	s2, 5		# size 
	la	s3, arrays	 
loop:	#do... 
	li	a0, 12 
	jal	malloc		# get memory for the next node 
	mv	s4, a0 
	li	a0, 20		 
	jal 	malloc		# get memory for this array 
	 
	sw	a0, 0(s4)	# node->arr = malloc 
	lw	a0, 0(s4) 
	mv	a1, s3 
	jal	fillArray	# copy ints over to node->arr 
	 
	sw	s2, 4(s4)	# node->size = size (4) 
	sw 	s0, 8(s4)     # node-> next = previously created node 
	 
	add	s0, x0, s4	# last = node 
	addi	s1, s1, 1	# i++ 
	addi	s3, s3, 20	# s3 points at next set of ints 	
	li t6 5
	bne	s1, t6, loop	# ... while i!= 5 
	mv	a0, s4 
	lw	ra, 0(sp) 
	addi	sp, sp, 4 
	jr	ra 
	 
fillArray:	lw	t0, 0(a1)	#t0 gets array element 
	sw	t0, 0(a0)	#node->arr gets array element 
	lw	t0, 4(a1) 
	sw	t0, 4(a0) 
	lw	t0, 8(a1) 
	sw	t0, 8(a0) 
	lw	t0, 12(a1) 
	sw	t0, 12(a0) 
	lw	t0, 16(a1) 
	sw	t0, 16(a0) 
	jr	ra 
 
print_list: 
	bne	a0, x0, printMeAndRecurse 
	jr	ra 		# nothing to print 
printMeAndRecurse: 
	mv	t0, a0	# t0 gets address of current node 
	lw	t3, 0(a0)	# t3 gets array of current node 
	li	t1, 0		# t1 is index into array 
printLoop: 
	slli	t2, t1, 2 
	add	t4, t3, t2 
	lw	a1, 0(t4)	# a0 gets value in current node's array at index t1 
	li	a0, 1		# preparte for print integer ecall 
	ecall 
	li	a1, ' '	# a0 gets address of string containing space 
	li	a0, 11		# prepare for print string ecall 
	ecall 
	addi	t1, t1, 1 
  	li t6 5
	bne	t1, t6, printLoop # ... while i!= 5 
	li	a1, '\n'	 
	li	a0, 11 
	ecall 
	lw	a0, 8(t0)	# a0 gets address of next node 
	j	print_list	# recurse. We don't have to use jal because we already have where we want to return to in ra 
 
print_newline: 
	li	a1, '\n' 
	li	a0, 11 
	ecall 
	jr	ra 
 
malloc: 
	mv a1, a0 # Move a0 into a1 so that we can do the syscall correctly
	li	a0, 9 
	ecall 
	jr	ra

Exercise 2

Modified Code:

.data
neg3:	.asciiz "f(-3) should be 6, and it is: "
neg2:	.asciiz "f(-2) should be 61, and it is: "
neg1:	.asciiz "f(-1) should be 17, and it is: "
zero:	.asciiz "f(0) should be -38, and it is: "
pos1:	.asciiz "f(1) should be 19, and it is: "
pos2:	.asciiz "f(2) should be 42, and it is: "
pos3:	.asciiz "f(3) should be 5, and it is: "

output:	.word	6, 61, 17, -38, 19, 42, 5
.text
main:
	la	a0, neg3
	jal	print_str
	li	a0, -3
	jal	f		# evaluate f(-3); should be 6
	jal	print_int
	jal	print_newline
	
	la	a0, neg2
	jal	print_str
	li	a0, -2
	jal	f		# evaluate f(-2); should be 61
	jal	print_int
	jal	print_newline
	
	la	a0, neg1
	jal	print_str
	li	a0, -1
	jal	f		# evaluate f(-1); should be 17
	jal	print_int
	jal	print_newline
	
	la	a0, zero
	jal	print_str
	li	a0, 0
	jal	f		# evaluate f(0); should be -38
	jal	print_int
	jal	print_newline
	
	la	a0, pos1
	jal	print_str
	li	a0, 1
	jal	f		# evaluate f(1); should be 19
	jal	print_int
	jal	print_newline
	
	la	a0, pos2
	jal	print_str
	li	a0, 2
	jal	f		# evaluate f(2); should be 42
	jal	print_int
	jal	print_newline
	
	la	a0, pos3
	jal	print_str
	li	a0, 3
	jal	f		# evaluate C(4,0); should be 5
	jal	print_int
	jal	print_newline

	li	a0, 10
	ecall

# calculate f(a0)
f:
	la	t0, output	# Hmm... why might this be a good idea?
	addi t1, a0, 3
	slli t1, t1, 2	
    	add t1, t0, t1
    	lw a0, 0(t1)
	jr	ra		# Always remember to jr ra after your function!
  
print_int:
	mv a1, a0
	li	a0, 1
	ecall
	jr	ra

print_str:
	mv a1, a0
	li	a0, 4
	ecall
	jr	ra
	
print_newline:
	li	a1, '\n'
	li	a0, 11
	ecall
	jr	ra


Exercise 3

Main Function:

000000000001019c <main>:
	1019c: 1101			addisp,sp,-32
	1091e: ec22			sd s0,24(sp)
	101a0: 1000			addi s0,sp,32
	101a2: 4785			li a5,1
	101a4: fef42623		sw a5,-20(s0)
	101a8: 478d			1i a5, 3
	101aa: fef42423		lw a4,-20(s0)
	101ae: fec42703		lw a4,-20(s0)
	101b2: fe842783		lw a5,-24(s0)
	101b6: 9fb9			addw a5, a5, a4
	101b8: 2781			sext.w a5, a5
	101ba: 853e			mv a0,a5
	101bc: 6462			ld s0,24(sp)
	101be: 6105			addisp,sp, 32
	101c0: 8082			ret 

Dump Function:

a.out:     file format elf64-littleriscv
Disassembly of section .text:

00000000000100b0 <_start>:
   100b0:	00002197          	auipc	gp,0x2
   100b4:	33018193          	addi	gp,gp,816 # 123e0 <__global_pointer$>
   100b8:	81818513          	addi	a0,gp,-2024 # 11bf8 <_edata>
   100bc:	85018613          	addi	a2,gp,-1968 # 11c30 <_end>
   100c0:	8e09                	sub	a2,a2,a0
   100c2:	4581                	li	a1,0
   100c4:	1cc000ef          	jal	ra,10290 <memset>
   100c8:	00000517          	auipc	a0,0x0
   100cc:	12650513          	addi	a0,a0,294 # 101ee <__libc_fini_array>
   100d0:	0f2000ef          	jal	ra,101c2 <atexit>
   100d4:	152000ef          	jal	ra,10226 <__libc_init_array>
   100d8:	4502                	lw	a0,0(sp)
   100da:	002c                	addi	a1,sp,8
   100dc:	4601                	li	a2,0
   100de:	0be000ef          	jal	ra,1019c <main>
   100e2:	0ec0006f          	j	101ce <exit>

00000000000100e6 <_fini>:
   100e6:	8082                	ret

00000000000100e8 <deregister_tm_clones>:
   100e8:	6549                	lui	a0,0x12
   100ea:	67c9                	lui	a5,0x12
   100ec:	be050713          	addi	a4,a0,-1056 # 11be0 <_global_impure_ptr>
   100f0:	be078793          	addi	a5,a5,-1056 # 11be0 <_global_impure_ptr>
   100f4:	00e78b63          	beq	a5,a4,1010a <deregister_tm_clones+0x22>
   100f8:	00000337          	lui	t1,0x0
   100fc:	00030313          	mv	t1,t1
   10100:	00030563          	beqz	t1,1010a <deregister_tm_clones+0x22>
   10104:	be050513          	addi	a0,a0,-1056
   10108:	8302                	jr	t1
   1010a:	8082                	ret

000000000001010c <register_tm_clones>:
   1010c:	67c9                	lui	a5,0x12
   1010e:	6549                	lui	a0,0x12
   10110:	be078593          	addi	a1,a5,-1056 # 11be0 <_global_impure_ptr>
   10114:	be050793          	addi	a5,a0,-1056 # 11be0 <_global_impure_ptr>
   10118:	8d9d                	sub	a1,a1,a5
   1011a:	858d                	srai	a1,a1,0x3
   1011c:	4789                	li	a5,2
   1011e:	02f5c5b3          	div	a1,a1,a5
   10122:	c991                	beqz	a1,10136 <register_tm_clones+0x2a>
   10124:	00000337          	lui	t1,0x0
   10128:	00030313          	mv	t1,t1
   1012c:	00030563          	beqz	t1,10136 <register_tm_clones+0x2a>
   10130:	be050513          	addi	a0,a0,-1056
   10134:	8302                	jr	t1
   10136:	8082                	ret

0000000000010138 <__do_global_dtors_aux>:
   10138:	8181c703          	lbu	a4,-2024(gp) # 11bf8 <_edata>
   1013c:	eb15                	bnez	a4,10170 <__do_global_dtors_aux+0x38>
   1013e:	1141                	addi	sp,sp,-16
   10140:	e022                	sd	s0,0(sp)
   10142:	e406                	sd	ra,8(sp)
   10144:	843e                	mv	s0,a5
   10146:	fa3ff0ef          	jal	ra,100e8 <deregister_tm_clones>
   1014a:	000007b7          	lui	a5,0x0
   1014e:	00078793          	mv	a5,a5
   10152:	cb81                	beqz	a5,10162 <__do_global_dtors_aux+0x2a>
   10154:	6545                	lui	a0,0x11
   10156:	48450513          	addi	a0,a0,1156 # 11484 <__FRAME_END__>
   1015a:	ffff0097          	auipc	ra,0xffff0
   1015e:	ea6080e7          	jalr	-346(ra) # 0 <_start-0x100b0>
   10162:	4785                	li	a5,1
   10164:	80f18c23          	sb	a5,-2024(gp) # 11bf8 <_edata>
   10168:	60a2                	ld	ra,8(sp)
   1016a:	6402                	ld	s0,0(sp)
   1016c:	0141                	addi	sp,sp,16
   1016e:	8082                	ret
   10170:	8082                	ret

0000000000010172 <frame_dummy>:
   10172:	000007b7          	lui	a5,0x0
   10176:	00078793          	mv	a5,a5
   1017a:	cf99                	beqz	a5,10198 <frame_dummy+0x26>
   1017c:	65c9                	lui	a1,0x12
   1017e:	6545                	lui	a0,0x11
   10180:	1141                	addi	sp,sp,-16
   10182:	c0058593          	addi	a1,a1,-1024 # 11c00 <object.5471>
   10186:	48450513          	addi	a0,a0,1156 # 11484 <__FRAME_END__>
   1018a:	e406                	sd	ra,8(sp)
   1018c:	ffff0097          	auipc	ra,0xffff0
   10190:	e74080e7          	jalr	-396(ra) # 0 <_start-0x100b0>
   10194:	60a2                	ld	ra,8(sp)
   10196:	0141                	addi	sp,sp,16
   10198:	f75ff06f          	j	1010c <register_tm_clones>

000000000001019c <main>:
   1019c:	1101                	addi	sp,sp,-32
   1019e:	ec22                	sd	s0,24(sp)
   101a0:	1000                	addi	s0,sp,32
   101a2:	4785                	li	a5,1
   101a4:	fef42623          	sw	a5,-20(s0)
   101a8:	478d                	li	a5,3
   101aa:	fef42423          	sw	a5,-24(s0)
   101ae:	fec42703          	lw	a4,-20(s0)
   101b2:	fe842783          	lw	a5,-24(s0)
   101b6:	9fb9                	addw	a5,a5,a4
   101b8:	2781                	sext.w	a5,a5
   101ba:	853e                	mv	a0,a5
   101bc:	6462                	ld	s0,24(sp)
   101be:	6105                	addi	sp,sp,32
   101c0:	8082                	ret

00000000000101c2 <atexit>:
   101c2:	85aa                	mv	a1,a0
   101c4:	4681                	li	a3,0
   101c6:	4601                	li	a2,0
   101c8:	4501                	li	a0,0
   101ca:	1700006f          	j	1033a <__register_exitproc>

00000000000101ce <exit>:
   101ce:	1141                	addi	sp,sp,-16
   101d0:	4581                	li	a1,0
   101d2:	e022                	sd	s0,0(sp)
   101d4:	e406                	sd	ra,8(sp)
   101d6:	842a                	mv	s0,a0
   101d8:	1c8000ef          	jal	ra,103a0 <__call_exitprocs>
   101dc:	67c9                	lui	a5,0x12
   101de:	be07b503          	ld	a0,-1056(a5) # 11be0 <_global_impure_ptr>
   101e2:	6d3c                	ld	a5,88(a0)
   101e4:	c391                	beqz	a5,101e8 <exit+0x1a>
   101e6:	9782                	jalr	a5
   101e8:	8522                	mv	a0,s0
   101ea:	266000ef          	jal	ra,10450 <_exit>

00000000000101ee <__libc_fini_array>:
   101ee:	1101                	addi	sp,sp,-32
   101f0:	67c5                	lui	a5,0x11
   101f2:	e822                	sd	s0,16(sp)
   101f4:	6445                	lui	s0,0x11
   101f6:	49078713          	addi	a4,a5,1168 # 11490 <__init_array_end>
   101fa:	49840413          	addi	s0,s0,1176 # 11498 <__fini_array_end>
   101fe:	8c19                	sub	s0,s0,a4
   10200:	e426                	sd	s1,8(sp)
   10202:	ec06                	sd	ra,24(sp)
   10204:	840d                	srai	s0,s0,0x3
   10206:	49078493          	addi	s1,a5,1168
   1020a:	e419                	bnez	s0,10218 <__libc_fini_array+0x2a>
   1020c:	6442                	ld	s0,16(sp)
   1020e:	60e2                	ld	ra,24(sp)
   10210:	64a2                	ld	s1,8(sp)
   10212:	6105                	addi	sp,sp,32
   10214:	ed3ff06f          	j	100e6 <_fini>
   10218:	147d                	addi	s0,s0,-1
   1021a:	00341793          	slli	a5,s0,0x3
   1021e:	97a6                	add	a5,a5,s1
   10220:	639c                	ld	a5,0(a5)
   10222:	9782                	jalr	a5
   10224:	b7dd                	j	1020a <__libc_fini_array+0x1c>

0000000000010226 <__libc_init_array>:
   10226:	1101                	addi	sp,sp,-32
   10228:	67c5                	lui	a5,0x11
   1022a:	e822                	sd	s0,16(sp)
   1022c:	6445                	lui	s0,0x11
   1022e:	48878713          	addi	a4,a5,1160 # 11488 <__frame_dummy_init_array_entry>
   10232:	48840413          	addi	s0,s0,1160 # 11488 <__frame_dummy_init_array_entry>
   10236:	8c19                	sub	s0,s0,a4
   10238:	e426                	sd	s1,8(sp)
   1023a:	e04a                	sd	s2,0(sp)
   1023c:	ec06                	sd	ra,24(sp)
   1023e:	840d                	srai	s0,s0,0x3
   10240:	4481                	li	s1,0
   10242:	48878913          	addi	s2,a5,1160
   10246:	02849763          	bne	s1,s0,10274 <__libc_init_array+0x4e>
   1024a:	e9dff0ef          	jal	ra,100e6 <_fini>
   1024e:	67c5                	lui	a5,0x11
   10250:	6445                	lui	s0,0x11
   10252:	48878713          	addi	a4,a5,1160 # 11488 <__frame_dummy_init_array_entry>
   10256:	49040413          	addi	s0,s0,1168 # 11490 <__init_array_end>
   1025a:	8c19                	sub	s0,s0,a4
   1025c:	840d                	srai	s0,s0,0x3
   1025e:	4481                	li	s1,0
   10260:	48878913          	addi	s2,a5,1160
   10264:	00849f63          	bne	s1,s0,10282 <__libc_init_array+0x5c>
   10268:	60e2                	ld	ra,24(sp)
   1026a:	6442                	ld	s0,16(sp)
   1026c:	64a2                	ld	s1,8(sp)
   1026e:	6902                	ld	s2,0(sp)
   10270:	6105                	addi	sp,sp,32
   10272:	8082                	ret
   10274:	00349793          	slli	a5,s1,0x3
   10278:	97ca                	add	a5,a5,s2
   1027a:	639c                	ld	a5,0(a5)
   1027c:	0485                	addi	s1,s1,1
   1027e:	9782                	jalr	a5
   10280:	b7d9                	j	10246 <__libc_init_array+0x20>
   10282:	00349793          	slli	a5,s1,0x3
   10286:	97ca                	add	a5,a5,s2
   10288:	639c                	ld	a5,0(a5)
   1028a:	0485                	addi	s1,s1,1
   1028c:	9782                	jalr	a5
   1028e:	bfd9                	j	10264 <__libc_init_array+0x3e>

0000000000010290 <memset>:
   10290:	433d                	li	t1,15
   10292:	872a                	mv	a4,a0
   10294:	02c37163          	bleu	a2,t1,102b6 <memset+0x26>
   10298:	00f77793          	andi	a5,a4,15
   1029c:	e3c1                	bnez	a5,1031c <memset+0x8c>
   1029e:	e1bd                	bnez	a1,10304 <memset+0x74>
   102a0:	ff067693          	andi	a3,a2,-16
   102a4:	8a3d                	andi	a2,a2,15
   102a6:	96ba                	add	a3,a3,a4
   102a8:	e30c                	sd	a1,0(a4)
   102aa:	e70c                	sd	a1,8(a4)
   102ac:	0741                	addi	a4,a4,16
   102ae:	fed76de3          	bltu	a4,a3,102a8 <memset+0x18>
   102b2:	e211                	bnez	a2,102b6 <memset+0x26>
   102b4:	8082                	ret
   102b6:	40c306b3          	sub	a3,t1,a2
   102ba:	068a                	slli	a3,a3,0x2
   102bc:	00000297          	auipc	t0,0x0
   102c0:	9696                	add	a3,a3,t0
   102c2:	00a68067          	jr	10(a3)
   102c6:	00b70723          	sb	a1,14(a4)
   102ca:	00b706a3          	sb	a1,13(a4)
   102ce:	00b70623          	sb	a1,12(a4)
   102d2:	00b705a3          	sb	a1,11(a4)
   102d6:	00b70523          	sb	a1,10(a4)
   102da:	00b704a3          	sb	a1,9(a4)
   102de:	00b70423          	sb	a1,8(a4)
   102e2:	00b703a3          	sb	a1,7(a4)
   102e6:	00b70323          	sb	a1,6(a4)
   102ea:	00b702a3          	sb	a1,5(a4)
   102ee:	00b70223          	sb	a1,4(a4)
   102f2:	00b701a3          	sb	a1,3(a4)
   102f6:	00b70123          	sb	a1,2(a4)
   102fa:	00b700a3          	sb	a1,1(a4)
   102fe:	00b70023          	sb	a1,0(a4)
   10302:	8082                	ret
   10304:	0ff5f593          	andi	a1,a1,255
   10308:	00859693          	slli	a3,a1,0x8
   1030c:	8dd5                	or	a1,a1,a3
   1030e:	01059693          	slli	a3,a1,0x10
   10312:	8dd5                	or	a1,a1,a3
   10314:	02059693          	slli	a3,a1,0x20
   10318:	8dd5                	or	a1,a1,a3
   1031a:	b759                	j	102a0 <memset+0x10>
   1031c:	00279693          	slli	a3,a5,0x2
   10320:	00000297          	auipc	t0,0x0
   10324:	9696                	add	a3,a3,t0
   10326:	8286                	mv	t0,ra
   10328:	fa2680e7          	jalr	-94(a3)
   1032c:	8096                	mv	ra,t0
   1032e:	17c1                	addi	a5,a5,-16
   10330:	8f1d                	sub	a4,a4,a5
   10332:	963e                	add	a2,a2,a5
   10334:	f8c371e3          	bleu	a2,t1,102b6 <memset+0x26>
   10338:	b79d                	j	1029e <memset+0xe>

000000000001033a <__register_exitproc>:
   1033a:	67c9                	lui	a5,0x12
   1033c:	be07b703          	ld	a4,-1056(a5) # 11be0 <_global_impure_ptr>
   10340:	832a                	mv	t1,a0
   10342:	1f873783          	ld	a5,504(a4)
   10346:	e789                	bnez	a5,10350 <__register_exitproc+0x16>
   10348:	20070793          	addi	a5,a4,512
   1034c:	1ef73c23          	sd	a5,504(a4)
   10350:	4798                	lw	a4,8(a5)
   10352:	487d                	li	a6,31
   10354:	557d                	li	a0,-1
   10356:	04e84463          	blt	a6,a4,1039e <__register_exitproc+0x64>
   1035a:	02030a63          	beqz	t1,1038e <__register_exitproc+0x54>
   1035e:	00371813          	slli	a6,a4,0x3
   10362:	983e                	add	a6,a6,a5
   10364:	10c83823          	sd	a2,272(a6)
   10368:	3107a883          	lw	a7,784(a5)
   1036c:	4605                	li	a2,1
   1036e:	00e6163b          	sllw	a2,a2,a4
   10372:	00c8e8b3          	or	a7,a7,a2
   10376:	3117a823          	sw	a7,784(a5)
   1037a:	20d83823          	sd	a3,528(a6)
   1037e:	4689                	li	a3,2
   10380:	00d31763          	bne	t1,a3,1038e <__register_exitproc+0x54>
   10384:	3147a683          	lw	a3,788(a5)
   10388:	8e55                	or	a2,a2,a3
   1038a:	30c7aa23          	sw	a2,788(a5)
   1038e:	0017069b          	addiw	a3,a4,1
   10392:	0709                	addi	a4,a4,2
   10394:	070e                	slli	a4,a4,0x3
   10396:	c794                	sw	a3,8(a5)
   10398:	97ba                	add	a5,a5,a4
   1039a:	e38c                	sd	a1,0(a5)
   1039c:	4501                	li	a0,0
   1039e:	8082                	ret

00000000000103a0 <__call_exitprocs>:
   103a0:	715d                	addi	sp,sp,-80
   103a2:	67c9                	lui	a5,0x12
   103a4:	f44e                	sd	s3,40(sp)
   103a6:	be07b983          	ld	s3,-1056(a5) # 11be0 <_global_impure_ptr>
   103aa:	f052                	sd	s4,32(sp)
   103ac:	ec56                	sd	s5,24(sp)
   103ae:	e85a                	sd	s6,16(sp)
   103b0:	e486                	sd	ra,72(sp)
   103b2:	e0a2                	sd	s0,64(sp)
   103b4:	fc26                	sd	s1,56(sp)
   103b6:	f84a                	sd	s2,48(sp)
   103b8:	e45e                	sd	s7,8(sp)
   103ba:	8aaa                	mv	s5,a0
   103bc:	8a2e                	mv	s4,a1
   103be:	4b05                	li	s6,1
   103c0:	1f89b483          	ld	s1,504(s3)
   103c4:	c881                	beqz	s1,103d4 <__call_exitprocs+0x34>
   103c6:	4480                	lw	s0,8(s1)
   103c8:	fff4091b          	addiw	s2,s0,-1
   103cc:	040e                	slli	s0,s0,0x3
   103ce:	9426                	add	s0,s0,s1
   103d0:	00095d63          	bgez	s2,103ea <__call_exitprocs+0x4a>
   103d4:	60a6                	ld	ra,72(sp)
   103d6:	6406                	ld	s0,64(sp)
   103d8:	74e2                	ld	s1,56(sp)
   103da:	7942                	ld	s2,48(sp)
   103dc:	79a2                	ld	s3,40(sp)
   103de:	7a02                	ld	s4,32(sp)
   103e0:	6ae2                	ld	s5,24(sp)
   103e2:	6b42                	ld	s6,16(sp)
   103e4:	6ba2                	ld	s7,8(sp)
   103e6:	6161                	addi	sp,sp,80
   103e8:	8082                	ret
   103ea:	000a0963          	beqz	s4,103fc <__call_exitprocs+0x5c>
   103ee:	20843783          	ld	a5,520(s0)
   103f2:	01478563          	beq	a5,s4,103fc <__call_exitprocs+0x5c>
   103f6:	397d                	addiw	s2,s2,-1
   103f8:	1461                	addi	s0,s0,-8
   103fa:	bfd9                	j	103d0 <__call_exitprocs+0x30>
   103fc:	449c                	lw	a5,8(s1)
   103fe:	6414                	ld	a3,8(s0)
   10400:	37fd                	addiw	a5,a5,-1
   10402:	03279763          	bne	a5,s2,10430 <__call_exitprocs+0x90>
   10406:	0124a423          	sw	s2,8(s1)
   1040a:	d6f5                	beqz	a3,103f6 <__call_exitprocs+0x56>
   1040c:	3104a703          	lw	a4,784(s1)
   10410:	012b163b          	sllw	a2,s6,s2
   10414:	0084ab83          	lw	s7,8(s1)
