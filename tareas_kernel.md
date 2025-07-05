## Agregar tareas de kernel:

```C
tss_t tss_create_kernel_task(paddr_t code_start) {
  //dir. virtual de comienzo del codigo
  vaddr_t code_virt = &funcionQueRealizaLaTarea();
  uint32_t cr3 = mmu_init_kernel_task_dir(code_start, code_virt);
  //asignar valor inicial de la pila de la tarea
  vaddr_t stack = mmu_next_free_kernel_page();//Por tener id mapping en kernel no hace falta mapearla a una virtual porque virtual = fisica
  //Pido pagina de kernel para la pila de nivel cero
  vaddr_t stack0 = stack; //Por ser de nivel cero su pila y su pila de nivel cero son la misma.
  vaddr_t esp0 = stack0 + PAGE_SIZE; //La pila crece restando el esp0
  return (tss_t) {
    .cr3 = cr3,
    .esp = stack,
    .ebp = stack,
    .eip = code_virt,
    .cs = GDT_CODE_0_SEL,
    .ds = GDT_DATA_0_SEL,
    .es = GDT_DATA_0_SEL,
    .fs = GDT_DATA_0_SEL,
    .gs = GDT_DATA_0_SEL,
    .ss = GDT_DATA_0_SEL,
    .ss0 = GDT_DATA_0_SEL,
    .esp0 = esp0,
    .eflags = EFLAGS_IF,
  };
}

vaddr_t mmu_init_kernel_task_dir(paddr_t phy_start, vaddr_t code_virt) {
  vaddr_t task_dir = mmu_next_free_kernel_page(); //Por id map pese a que la función me da una dir fisica es igual a la dir virtual
  zero_page(task_dir); 
  /*Identity map en primeros 4MB así puedo acceder a kernel ante interrupción u otro requerimiento*/
  for(int i = 0; i < identity_mapping_end; i += PAGE_SIZE){
    mmu_map_page(task_dir, i, i, MMU_W |MMU_P);
  }
  /*Paginas codigo tarea*/
  mmu_map_page(task_dir, code_virt + PAGE_SIZE , phy_start + PAGE_SIZE, MMU_P);  
  /*Pagina stack*/ 
  mmu_map_page(task_dir, KERNEL_TASK_STACK_BASE - PAGE_SIZE, mmu_next_free_kernel_page(), MMU_W | MMU_P);
  return (vaddr_t)task_dir;
}

static int8_t create_kernel_task(tipo_e tipo) {
    size_t gdt_id;
    for (gdt_id = GDT_TSS_START; gdt_id < GDT_COUNT; gdt_id++) {
        if (gdt[gdt_id].p == 0) {
              break;
            }
          }
    kassert(gdt_id < GDT_COUNT, "No hay entradas disponibles en la GDT");

    int8_t task_id = sched_add_task(gdt_id << 3);
    tss_tasks[task_id] = tss_create_kernel_task(/*Dirección física donde la quieras ubicar, de 0x1E000 en adelante hasta 0x24000 que tenemos la pila del kernel*/);
    gdt[gdt_id] = tss_gdt_entry_for_task(&tss_tasks[task_id]);
    return task_id;
}
