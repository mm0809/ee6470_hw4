# ee6470_hw4

## How to run

### 1. copy hardware and software code
```shell
$ cp -r vp-basic-hw/* $EE6470/riscv-vp/vp/src/platform
$ cp -r sw-gaussian/* $EE6470/riscv-vp/sw
```

### 2. compile hardware part
```shell
$ cd $EE6470/riscv-vp/vp/build
$ cmake ..
$ make install
```

### 3. run the simulation
```shell
$ cd $EE6470/riscv-vp/sw/sw-gaussian
$ make sim
```

## Result

![](https://i.imgur.com/pHPoqKt.png)

## Code

### hardware part

First we need to change the algorithm in `do_filter`.
```cpp
void do_filter(){
    { wait(CLOCK_PERIOD, SC_NS); }
    while (true) {
      int i = 0;
      val_r = 0;
      val_g = 0;
      val_b = 0;
      for (unsigned int v = 0; v < MASK_Y; ++v) {
        for (unsigned int u = 0; u < MASK_X; ++u) {
          val_r += i_r.read() * mask[i][u][v];
          val_g += i_g.read() * mask[i][u][v];
          val_b += i_b.read() * mask[i][u][v];
          wait(CLOCK_PERIOD, SC_NS);
        }
      }
      val_r = val_r >> 4;
      val_g = val_g >> 4;
      val_b = val_b >> 4;
      wait(CLOCK_PERIOD, SC_NS);

      o_result_r.write((unsigned char)val_r);
      o_result_g.write((unsigned char)val_g);
      o_result_b.write((unsigned char)val_b);
    }
  }

```

The output also need modify. In the original version it only can output 1 Byte, but we need 3 Bytes bandwidth.
```cpp
    switch (cmd) {
      case tlm::TLM_READ_COMMAND:
        // cout << "READ" << endl;
        switch (addr) {
          case SOBEL_FILTER_RESULT_ADDR:
            buffer.uc[0] = o_result_r.read();
            buffer.uc[1] = o_result_g.read();
            buffer.uc[2] = o_result_b.read();
            break;
          default:
            std::cerr << "READ Error! SobelFilter::blocking_transport: address 0x"
                      << std::setfill('0') << std::setw(8) << std::hex << addr
                      << std::dec << " is not valid" << std::endl;
          }
        data_ptr[0] = buffer.uc[0];
        data_ptr[1] = buffer.uc[1];
        data_ptr[2] = buffer.uc[2];
        //data_ptr[3] = buffer.uc[3];
        break;
      case tlm::TLM_WRITE_COMMAND:
        // cout << "WRITE" << endl;
        switch (addr) {
          case SOBEL_FILTER_R_ADDR:
            i_r.write(data_ptr[0]);
            i_g.write(data_ptr[1]);
            i_b.write(data_ptr[2]);
            break;
          default:
            std::cerr << "WRITE Error! SobelFilter::blocking_transport: address 0x"
                      << std::setfill('0') << std::setw(8) << std::hex << addr
                      << std::dec << " is not valid" << std::endl;
        }
        break;
      case tlm::TLM_IGNORE_COMMAND:
        payload.set_response_status(tlm::TLM_GENERIC_ERROR_RESPONSE);
        return;
      default:
        payload.set_response_status(tlm::TLM_GENERIC_ERROR_RESPONSE);
        return;
      }
      payload.set_response_status(tlm::TLM_OK_RESPONSE); // Always OK
  }
```

### software part
In `main.cpp` we need to change the way of input (index i, j, u, v), if we don't change the output picture will be rotate and flip.
```cpp
int main(int argc, char *argv[]) {

  read_bmp("lena_std_short.bmp");
  printf("======================================\n");
  printf("\t  Reading from array\n");
  printf("======================================\n");
	printf(" input_rgb_raw_data_offset\t= %d\n", input_rgb_raw_data_offset);
	printf(" width\t\t\t\t= %d\n", width);
	printf(" height\t\t\t\t= %d\n", height);
	printf(" bytes_per_pixel\t\t= %d\n",bytes_per_pixel);
  printf("======================================\n");

  unsigned char  buffer[4] = {0};
  word data;
  int total;
  printf("Start processing...(%d, %d)\n", width, height);
  for(int i = 0; i < width; i++){
    for(int j = 0; j < height; j++){
      //printf("pixel (%d, %d); \n", i, j);
      for(int v = -1; v <= 1; v ++){
        for(int u = -1; u <= 1; u++){
          if((v + i) >= 0  &&  (v + i ) < width && (u + j) >= 0 && (u + j) < height ){
            buffer[0] = *(source_bitmap + bytes_per_pixel * ((i + v) * width + (j + u)) + 2);
            buffer[1] = *(source_bitmap + bytes_per_pixel * ((i + v) * width + (j + u)) + 1);
            buffer[2] = *(source_bitmap + bytes_per_pixel * ((i + v) * width + (j + u)) + 0);
            buffer[3] = 0;
          }else{
            buffer[0] = 0;
            buffer[1] = 0;
            buffer[2] = 0;
            buffer[3] = 0;
          }
          write_data_to_ACC(SOBELFILTER_START_ADDR, buffer, 4);
        }
      }
      read_data_from_ACC(SOBELFILTER_READ_ADDR, buffer, 4);

      memcpy(data.uc, buffer, 4);

      *(target_bitmap + bytes_per_pixel * (width * i + j) + 2) = data.uc[0];
      *(target_bitmap + bytes_per_pixel * (width * i + j) + 1) = data.uc[1];
      *(target_bitmap + bytes_per_pixel * (width * i + j) + 0) = data.uc[2];

    }
  }
  write_bmp("lena_std_out.bmp");
}
```
