---
layout: post
title: "使用rawsock构造IPv6首部"
date: 2016-12-09 
tags: Linux编程  
---

### 发送端（定制ipv6首部）
    #include <stdio.h>
    #include <stdlib.h>
    #include <stdarg.h>
    #include <string.h>
    #include <time.h>
    #include <sys/types.h>
    
    #include <unistd.h>
    #include <errno.h>
    #include <fcntl.h>
    #include <signal.h>
    #include <termios.h>
    #include <sys/ioctl.h>
    #include <sys/time.h>
    
    #include <sys/socket.h>
    #include <netinet/in.h>
    #include <netinet/ip.h>
    #include <netinet/ip6.h>
    #include <netinet/udp.h>
    #include <linux/if.h>
    #include <linux/if_ether.h>
    #include <linux/if_packet.h>
    #include <arpa/inet.h>
    #include <netdb.h>
    #include <pthread.h>
    
    #include <err.h>
    
    #define IPV6_HEADER_LEN      sizeof(struct ip6_hdr)
    #define UDP_HEADER_LEN       sizeof(struct udphdr)
    #define IPV6_UDP_HEADER_LEN    IPV6_HEADER_LEN+UDP_HEADER_LEN
    
    // Computing the internet checksum (RFC 1071).
    // Note that the internet checksum does not preclude collisions.
    uint16_t
    checksum (uint16_t *addr, int len)
    {
        int count = len;
        register uint32_t sum = 0;
        uint16_t answer = 0;
    
        // Sum up 2-byte values until none or only one byte left.
        while (count > 1) {
            sum += *(addr++);
            count -= 2;
        }
    
        // Add left-over byte, if any.
        if (count > 0) {
            sum += *(uint8_t *) addr;
        }
    
        // Fold 32-bit sum into 16 bits; we lose information by doing this,
        // increasing the chances of a collision.
        // sum = (lower 16 bits) + (upper 16 bits shifted right 16 bits)
        while (sum >> 16) {
            sum = (sum & 0xffff) + (sum >> 16);
        }
    
        // Checksum is one's compliment of sum.
        answer = ~sum;
        return (answer);
    }
    
    // Build IPv6 UDP pseudo-header and call checksum function (Section 8.1 of RFC 2460).
    uint16_t udp6_checksum (struct ip6_hdr iphdr, struct udphdr udphdr, uint8_t *payload, int payloadlen)
    {
        char buf[IP_MAXPACKET];
        char *ptr;
        int chksumlen = 0;
        int i;
    
        ptr = &buf[0];  // ptr points to beginning of buffer buf
    
        // Copy source IP address into buf (128 bits)
        memcpy (ptr, &iphdr.ip6_src.s6_addr, sizeof (iphdr.ip6_src.s6_addr));
        ptr += sizeof (iphdr.ip6_src.s6_addr);
        chksumlen += sizeof (iphdr.ip6_src.s6_addr);
    
        // Copy destination IP address into buf (128 bits)
        memcpy (ptr, &iphdr.ip6_dst.s6_addr, sizeof (iphdr.ip6_dst.s6_addr));
        ptr += sizeof (iphdr.ip6_dst.s6_addr);
        chksumlen += sizeof (iphdr.ip6_dst.s6_addr);
        
        // Copy UDP length into buf (32 bits)
        memcpy (ptr, &udphdr.uh_ulen, sizeof (udphdr.uh_ulen));
        ptr += sizeof (udphdr.uh_ulen);
        chksumlen += sizeof (udphdr.uh_ulen);
        
        // Copy zero field to buf (24 bits)
        *ptr = 0; ptr++;
        *ptr = 0; ptr++;
        *ptr = 0; ptr++;
        chksumlen += 3;
        
        // Copy next header field to buf (8 bits)
        memcpy (ptr, &iphdr.ip6_nxt, sizeof (iphdr.ip6_nxt));
        ptr += sizeof (iphdr.ip6_nxt);
        chksumlen += sizeof (iphdr.ip6_nxt);
        
        // Copy UDP source port to buf (16 bits)
        memcpy (ptr, &udphdr.uh_sport, sizeof (udphdr.uh_sport));
        ptr += sizeof (udphdr.uh_sport);
        chksumlen += sizeof (udphdr.uh_sport);
        
        // Copy UDP destination port to buf (16 bits)
        memcpy (ptr, &udphdr.uh_dport, sizeof (udphdr.uh_dport));
        ptr += sizeof (udphdr.uh_dport);
        chksumlen += sizeof (udphdr.uh_dport);
        
        // Copy UDP length again to buf (16 bits)
        memcpy (ptr, &udphdr.uh_ulen, sizeof (udphdr.uh_ulen));
        ptr += sizeof (udphdr.uh_ulen);
        chksumlen += sizeof (udphdr.uh_ulen);
        
        // Copy UDP checksum to buf (16 bits)
        // Zero, since we don't know it yet
        *ptr = 0; ptr++;
        *ptr = 0; ptr++;
        chksumlen += 2;
        
        // Copy payload to buf
        memcpy (ptr, payload, payloadlen * sizeof (uint8_t));
        ptr += payloadlen;
        chksumlen += payloadlen;
        
        // Pad to the next 16-bit boundary
        for (i=0; i<payloadlen%2; i++, ptr++) {
            *ptr = 0;
            ptr++;
            chksumlen++;
        }
        
        return checksum ((uint16_t *) buf, chksumlen);
    }

    // Allocate memory for an array of chars.
    char *allocate_strmem (int len)
    {
        void *tmp;
    
        if (len <= 0) {
            fprintf (stderr, "ERROR: Cannot allocate memory because len = %i in allocate_strmem().\n", len);
            exit (EXIT_FAILURE);
        }
        
        tmp = (char *) malloc (len * sizeof (char));
        if (tmp != NULL) {
            memset (tmp, 0, len * sizeof (char));
            return (tmp);
        } 
        else {
            fprintf (stderr, "ERROR: Cannot allocate memory for array allocate_strmem().\n");
            exit (EXIT_FAILURE);
        }
    }
    
    
    // Allocate memory for an array of unsigned chars.
    uint8_t *allocate_ustrmem (int len)
    {
        void *tmp;
        
        if (len <= 0) {
            fprintf (stderr, "ERROR: Cannot allocate memory because len = %i in allocate_ustrmem().\n", len);
            exit (EXIT_FAILURE);
        }
        
        tmp = (uint8_t *) malloc (len * sizeof (uint8_t));
        if (tmp != NULL) {
            memset (tmp, 0, len * sizeof (uint8_t));
            return (tmp);
        } 
        else {
            fprintf (stderr, "ERROR: Cannot allocate memory for array allocate_ustrmem().\n");
            exit (EXIT_FAILURE);
        }
    }
    
    void creat_sockfd6(int *sockfd, struct sockaddr_in6 *local, int portnum){
	    *sockfd = socket(PF_PACKET, SOCK_DGRAM, htons(ETH_P_IPV6));
	    if(*sockfd < 0){
		    perror("socket");
		    exit(EXIT_FAILURE);
	    }
    }
    
    int main(int argc, char *argv[])
    {
        struct sockaddr_in6 local_addr6;
        creat_sockfd6(&sdn_fd, &local_addr6, SDN_SRC_PORT6);
        
        int sd, data_len, status, i;
        char *interface, *src_ip, *dst_ip;
        char ret_len;
        char sdn_buf[ IPV6_UDP_HEADER_LEN + data_len];
        uint8_t *src_mac, *dst_mac, *data;
        struct addrinfo hints, *res;
        struct ip6_hdr* ipv6_header;
        struct udphdr* udp_header;
        struct sockaddr_in6 *dst_addr;
        struct sockaddr_ll device;
        struct ifreq ifr;
        void *tmp;
        
        src_mac = allocate_ustrmem (6);
        dst_mac = allocate_ustrmem (6);
        data = allocate_ustrmem (IP_MAXPACKET);
        src_ip = allocate_strmem (INET6_ADDRSTRLEN);
        dst_ip = allocate_strmem (INET6_ADDRSTRLEN);
        interface = allocate_strmem (INET6_ADDRSTRLEN);
        
        //interface to send packet through
        strcpy(interface, "br-lan");
        
        // Submit request for a socket descriptor to look up interface.
        if ((sd = socket (AF_INET6, SOCK_RAW, IPPROTO_RAW)) < 0) {
        	perror ("socket() failed to get socket descriptor for using ioctl() ");
        	exit (EXIT_FAILURE);
        }
        
        // Use ioctl() to look up interface name and get its MAC address.
        memset (&ifr, 0, sizeof (ifr));
        snprintf (ifr.ifr_name, sizeof (ifr.ifr_name), "%s", interface);
        if (ioctl (sd, SIOCGIFHWADDR, &ifr) < 0) {
        	perror ("ioctl() failed to get source MAC address ");
        	exit (EXIT_FAILURE);
        }
        close (sd);
        
        // Copy source MAC address.
        memcpy (src_mac, ifr.ifr_hwaddr.sa_data, 6 * sizeof (uint8_t));
        
        // Report source MAC address to stdout.
        printf ("MAC address for interface %s is ", interface);
        for (i=0; i<5; i++) {
          printf ("%02x:", src_mac[i]);
        }
        printf ("%02x\n", src_mac[5]);
        
        // Find interface index from interface name and store index in
        // struct sockaddr_ll device, which will be used as an argument of sendto().
        memset (&device, 0, sizeof (device));
        if ((device.sll_ifindex = if_nametoindex (interface)) == 0) {
          perror ("if_nametoindex() failed to obtain interface index ");
          exit (EXIT_FAILURE);
        }
        
        //set destination MAC address
        dst_mac[0] = 0x7a;
        dst_mac[1] = 0x20;
        dst_mac[2] = 0x04;
        dst_mac[3] = 0x04;
        dst_mac[4] = 0xc2;
        dst_mac[5] = 0x9c;
        
        // Source IPv6 address: you need to fill this out
        strcpy (src_ip, "2016::10");
        // Destination URL or IPv6 address: you need to fill this out
         trcpy (dst_ip, "2016::101");
         
        // Fill out hints for getaddrinfo().
        memset (&hints, 0, sizeof (hints));
        hints.ai_family = AF_INET6;
        hints.ai_socktype = SOCK_STREAM;
        hints.ai_flags = hints.ai_flags | AI_CANONNAME;
        
        // Resolve target using getaddrinfo().
        if ((status = getaddrinfo (dst_ip, NULL, &hints, &res)) != 0) {
            fprintf (stderr, "getaddrinfo() failed: %s\n", gai_strerror (status));
            exit (EXIT_FAILURE);
        }
        dst_addr = (struct sockaddr_in6 *) res->ai_addr;
        tmp = &(dst_addr->sin6_addr);
        if (inet_ntop (AF_INET6, tmp, dst_ip, INET6_ADDRSTRLEN) == NULL) {
            status = errno;
        	fprintf (stderr, "inet_ntop() failed.\nError message: %s", strerror (status));
        	exit (EXIT_FAILURE);
        }
        freeaddrinfo (res);
        
        // Fill out sockaddr_ll.
        device.sll_family = AF_PACKET;
        device.sll_protocol = htons (ETH_P_IPV6);
        memcpy (device.sll_addr, dst_mac, 6 * sizeof (uint8_t));
        device.sll_halen = 6;
        
        // UDP data
        data_len = 4;
        data[0] = 'T';
        data[1] = 'e';
        data[2] = 's';
        data[3] = 't';
        
        //form ipv6 header
        ipv6_header = (struct ip6_hdr *)malloc(IPV6_HEADER_LEN);
        ipv6_header->ip6_ctlun.ip6_un1.ip6_un1_flow = htonl(0x60000001);	 //流标签位置1
        ipv6_header->ip6_ctlun.ip6_un1.ip6_un1_plen = htons(UDP_HEADER_LEN + data_len);
        ipv6_header->ip6_ctlun.ip6_un1.ip6_un1_nxt = IPPROTO_UDP;				  //next header:udp
        ipv6_header->ip6_ctlun.ip6_un1.ip6_un1_hlim = 0xff;
        inet_pton(AF_INET6,src_ip,&(ipv6_header->ip6_src));  
        inet_pton(AF_INET6,dst_ip,&(ipv6_header->ip6_dst));  
        
        //form udp header
        udp_header = (struct udphdr *)malloc(UDP_HEADER_LEN);
        udp_header->uh_sport = htons(SDN_SRC_PORT6);
        udp_header->uh_dport = htons(SDN_DST_PORT6);
        udp_header->uh_ulen = htons(UDP_HEADER_LEN + data_len);
        udp_header->uh_sum = udp6_checksum(*ipv6_header, *udp_header, data, data_len);
        
        //form packet
        bzero(sdn_buf, IPV6_UDP_HEADER_LEN + data_len);
        memcpy(sdn_buf, ipv6_header, IPV6_HEADER_LEN);
        memcpy(sdn_buf + IPV6_HEADER_LEN, udp_header, UDP_HEADER_LEN);
        memcpy(sdn_buf + IPV6_UDP_HEADER_LEN, data, data_len);
        
        ret_len = sendto(sdn_fd, sdn_buf, IPV6_UDP_HEADER_LEN + data_len, 0, (struct sockaddr*) &device, sizeof(device));
        if(ret_len > 0){
        	printf("send ok\n");
        }
        else{
        	printf("send fail\n");
        }
        
        // Free allocated memory.
        free (src_mac);
        free (dst_mac);
        free (data);
        free (interface);
        free (src_ip);
        free (dst_ip);
    }
      
### 接收端（普通的ipv6套接字接收udp报文）
    #include <stdio.h>  
    #include <stdlib.h>  
    #include <unistd.h>  
    #include <string.h>  
    #include <netinet/in.h>  
    #include <sys/socket.h>  
    #include <sys/types.h>  
      
    #define DEFAULT_IP "2016::10" //本地环回地址  
    #define DEFAULT_PORT 5228  
      
      
    int main(int argc, char *argv[])  
    {  
        int sock = -1;  
        int ret  = -1;  
        int addr_len = 0;  
        struct sockaddr_in6 server_addr;  
        struct sockaddr_in6 client_addr;  
          
        sock = socket(AF_INET6, SOCK_DGRAM, 0);  
        if (sock < 0) {  
            perror("Fail to socket");  
            exit(1);  
        }  
      
        memset(&server_addr, 0, sizeof(server_addr));  
        memset(&client_addr, 0, sizeof(client_addr));  
          
        server_addr.sin6_family = AF_INET6; //IPv6 协议簇  
        server_addr.sin6_port = htons(DEFAULT_PORT);  
        inet_pton(AF_INET6, "2016::101", &(server_addr.sin6_addr.s6_addr)); //地址转换函数，这是一个通用函数，在IPv4中也可以使用，另外还有一个inet_ntop(),用于相反操作  
      
        addr_len = sizeof(server_addr);  
      
        if (bind(sock, (struct sockaddr *)&server_addr, addr_len) < 0) {  
            perror("Fail to bind");  
            exit(1);  
        }  
      
        unsigned char buf[17];  
      
        while (1) {  
            addr_len = sizeof(client_addr);  
            ret = recvfrom(sock, buf, 17, 0, (struct sockaddr *)&client_addr, &addr_len);  
            if (ret < 0) {  
                perror("Fail to recvfrom");  
                exit(1);  
            }  
            int i=0;
            for(i=0; i<16; i++)
            {
                printf("%02x  ", buf[i]);
            }
            printf("%02x\n", buf[16]);
        }  
        return 0;  
    }
***
[参考资料0](http://stackoverflow.com/questions/31419727/how-to-send-modified-ipv6-packet-through-raw-socket)

[参考资料1](http://www.cnblogs.com/yuuyuu/p/5169931.html)

[参考资料2](http://blog.csdn.net/hepeng597/article/details/7803277)