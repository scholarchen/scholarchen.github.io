---
layout: post
title: getifadds系统调用不能使用
category: debug
tags: uclibc,netlink
---
## 问题现象：
在libc库 UCLIBC0.9.33.2版本 内核3.10
所有调用getifaddrs的函数都不能正常工作，导致功能出错
## 问题模拟复现：
通过查看getifaddrs发现是libc中的库函数，并且内容较多，编译libc调试不方便。将相关的函数调用全部复制为一个小的test文件getifaddrs.c。编译并调试
## 
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <ifaddrs.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>
#include <netdb.h>
#include <stdbool.h>
#include <assert.h>
#include <errno.h>
#if 0
#define IF_NAMESIZE 16
#ifndef __libc_use_alloca
#define __libc_use_alloca( x ) (x < __MAX_ALLOCA_CUTOFF)
#endif
struct netlink_res
{
	struct netlink_res	*next;
	struct nlmsghdr		*nlh;
	size_t			size;   /* Size of response.  */
	uint32_t		seq;    /* sequential number we used.  */
};

#define __MAX_ALLOCA_CUTOFF 65536
#define _STACK_GROWS_DOWN
#ifdef _STACK_GROWS_DOWN
#define extend_alloca( buf, len, newlen ) \
	(__typeof( buf ) )({ size_t __newlen = (newlen);			    \
			     char *__newbuf = alloca( __newlen );		      \
			     if ( __newbuf + __newlen == (char *) buf )		       \
				     len += __newlen;					   \
			     else						     \
				     len = __newlen;					   \
			     __newbuf; })
#elif defined _STACK_GROWS_UP
#define extend_alloca( buf, len, newlen ) \
	(__typeof( buf ) )({ size_t __newlen = (newlen);			    \
			     char *__newbuf = alloca( __newlen );		      \
			     char *__buf = (buf);				     \
			     if ( __buf + __newlen == __newbuf )		       \
			     {							   \
				     len += __newlen;					 \
				     __newbuf = __buf;					 \
			     }							   \
			     else						     \
				     len = __newlen;					   \
			     __newbuf; })
#else
#error unknown stack
#define extend_alloca( buf, len, newlen ) \
	alloca( ( (len) = (newlen) ) )
#endif

struct netlink_handle
{
	int			fd;             /* Netlink file descriptor.  */
	pid_t			pid;            /* Process ID.  */
	uint32_t		seq;            /* The sequence number we use currently.  */
	struct netlink_res	*nlm_list;      /* Pointer to list of responses.  */
	struct netlink_res	*end_ptr;       /* For faster append of new entries.  */
};
struct sockaddr_ll
{
	unsigned short int	sll_family;
	unsigned short int	sll_protocol;
	int			sll_ifindex;
	unsigned short int	sll_hatype;
	unsigned char		sll_pkttype;
	unsigned char		sll_halen;
	unsigned char		sll_addr[8];
};

struct ifaddrs_storage
{
	struct ifaddrs ifa;
	union
	{
		/* Save space for the biggest of the four used sockaddr types and
		 * avoid a lot of casts.  */
		struct sockaddr		sa;
		struct sockaddr_ll	sl;
		struct sockaddr_in	s4;
#ifdef __UCLIBC_HAS_IPV6__
		struct sockaddr_in6 s6;
#endif
	}	addr, netmask, broadaddr;
	char	name[IF_NAMESIZE + 1];
};


void
__netlink_free_handle( struct netlink_handle *h )
{
	struct netlink_res *ptr;

	ptr = h->nlm_list;
	while ( ptr != NULL )
	{
		struct netlink_res *tmpptr;

		tmpptr = ptr->next;
		free( ptr ); /* doesn't affect errno */
		ptr = tmpptr;
	}
}


static int
__netlink_sendreq( struct netlink_handle *h, int type )
{
	struct
	{
		struct nlmsghdr nlh;
		struct rtgenmsg g;
	}			req;
	struct sockaddr_nl	nladdr;

	if ( h->seq == 0 )
		h->seq = time( NULL );

	req.nlh.nlmsg_len	= sizeof(req);
	req.nlh.nlmsg_type	= type;
	req.nlh.nlmsg_flags	= NLM_F_ROOT | NLM_F_MATCH | NLM_F_REQUEST;
	req.nlh.nlmsg_pid	= 0;
	req.nlh.nlmsg_seq	= h->seq;
	req.g.rtgen_family	= AF_UNSPEC;

	memset( &nladdr, '\0', sizeof(nladdr) );
	nladdr.nl_family = AF_NETLINK;
	printf( "%s %d sendto h->fd %d h->seq %d , h->pid %d type%d\n", __FUNCTION__, __LINE__, h->fd, h->seq, h->pid ,type);

	return(sendto( h->fd, (void *) &req, sizeof(req), 0,
		       (struct sockaddr *) &nladdr,
		       sizeof(nladdr) ) );
}
#define TEMP_FAILURE_RETRY(expression) \
(__extension__	\
({ long int __result;	\
do __result = (long int) (expression);	\
while (__result == -1L && errno == EINTR);	\
__result; }))

int
__netlink_request( struct netlink_handle *h, int type )
{
	struct netlink_res	*nlm_next;
	struct netlink_res	**new_nlm_list;
	static volatile size_t	buf_size = 4096;
	char			*buf;
	struct sockaddr_nl	nladdr;
	struct nlmsghdr		*nlmh;
	ssize_t			read_len;
	bool			done		= false;
	bool			use_malloc	= false;

	if ( __netlink_sendreq( h, type ) < 0 )
	{
		printf( "%s %d\n", __FUNCTION__, __LINE__ );
		return(-1);
	}

	size_t this_buf_size = buf_size;
	if ( __libc_use_alloca( this_buf_size ) )
		buf = alloca( this_buf_size );
	else{
		buf = malloc( this_buf_size );
		if ( buf != NULL )
			use_malloc = true;
		else
			goto out_fail;
	}

	struct iovec iov = { buf, this_buf_size };

	if ( h->nlm_list != NULL )
		new_nlm_list = &h->end_ptr->next;
	else
		new_nlm_list = &h->nlm_list;

	while ( !done )
	{
		struct msghdr msg =
		{
			(void *) &nladdr, sizeof(nladdr),
			&iov,		  1,
			NULL,		  0,
			0
		};
	printf( "%s %d recv h->fd %d h->seq %d , h->pid %d type%d\n", __FUNCTION__, __LINE__, h->fd, h->seq, h->pid,type);
		read_len = TEMP_FAILURE_RETRY(recvmsg( h->fd, &msg, 0 ) );
		if ( read_len < 0 )
			goto out_fail;

		if ( nladdr.nl_pid != 0 )
			continue;

		if ( __builtin_expect( msg.msg_flags & MSG_TRUNC, 0 ) )
		{
			if ( this_buf_size >= SIZE_MAX / 2 )
				goto out_fail;

			nlm_next = *new_nlm_list;
			while ( nlm_next != NULL )
			{
				struct netlink_res *tmpptr;

				tmpptr = nlm_next->next;
				free( nlm_next );
				nlm_next = tmpptr;
			}
			*new_nlm_list = NULL;

			if ( __libc_use_alloca( 2 * this_buf_size ) )
				buf = extend_alloca( buf, this_buf_size, 2 * this_buf_size );
			else{
				this_buf_size *= 2;

				char *new_buf = realloc( use_malloc ? buf : NULL, this_buf_size );
				if ( new_buf == NULL )
					goto out_fail;
				new_buf = buf;

				use_malloc = true;
			}
			buf_size = this_buf_size;

			iov.iov_base	= buf;
			iov.iov_len	= this_buf_size;


			/* Increase sequence number, so that we can distinguish
			 * between old and new request messages.  */
			h->seq++;

			if ( __netlink_sendreq( h, type ) < 0 )
			{
				printf( "%s %d h->seq, h->pid\n", __FUNCTION__, __LINE__, h->seq, h->pid );
				goto out_fail;
			}

			continue;
		}

		size_t	count		= 0;
		size_t	remaining_len	= read_len;
		for ( nlmh = (struct nlmsghdr *) buf;
		      NLMSG_OK( nlmh, remaining_len );
		      nlmh = (struct nlmsghdr *) NLMSG_NEXT( nlmh, remaining_len ) )
		{
			printf( "%s %d recv h->fd %d h->seq %d ,nlmh->nlmsg_seq %d ,nlmh->nlmsg_pid %d, h->pid %d type%d\n", __FUNCTION__, __LINE__, h->fd, h->seq,nlmh->nlmsg_seq,nlmh->nlmsg_pid , h->pid,type);
			if ( (pid_t) nlmh->nlmsg_pid != h->pid
			     || nlmh->nlmsg_seq != h->seq )
				continue;
			++count;
			if ( nlmh->nlmsg_type == NLMSG_DONE )
			{
				/* We found the end, leave the loop.  */
				done = true;
				break;
			}
			if ( nlmh->nlmsg_type == NLMSG_ERROR )
			{
				struct nlmsgerr *nlerr = (struct nlmsgerr *) NLMSG_DATA( nlmh );
				if ( nlmh->nlmsg_len < NLMSG_LENGTH( sizeof(struct nlmsgerr) ) )
					errno = EIO;
				else
					errno = -nlerr->error;
				printf( "%s %d  errno%d\n", __FUNCTION__, __LINE__,errno);
				goto out_fail;
			}
		}


		/* If there was nothing with the expected nlmsg_pid and nlmsg_seq,
		 * there is no point to record it.  */
		if ( count == 0 )
			continue;

		nlm_next = (struct netlink_res *) malloc( sizeof(struct netlink_res)
							  + read_len );
		if ( nlm_next == NULL )
			goto out_fail;
		nlm_next->next	= NULL;
		nlm_next->nlh	= memcpy( nlm_next + 1, buf, read_len );
		nlm_next->size	= read_len;
		nlm_next->seq	= h->seq;
		if ( h->nlm_list == NULL )
			h->nlm_list = nlm_next;
		else
			h->end_ptr->next = nlm_next;
		h->end_ptr = nlm_next;
	}

	if ( use_malloc )
		free( buf );
	return(0);

out_fail:
	if ( use_malloc )
		free( buf );
	printf( "%s %d\n", __FUNCTION__, __LINE__ );
	return(-1);
}


void
__netlink_close( struct netlink_handle *h )
{
	/* Don't modify errno.  */
	int serrno = errno;
	close( h->fd );
}


/* Open a NETLINK socket.  */
int
__netlink_open( struct netlink_handle *h )
{
	struct sockaddr_nl nladdr;

	h->fd = socket( PF_NETLINK, SOCK_RAW, NETLINK_ROUTE );
	if ( h->fd < 0 )
		goto out;

	memset( &nladdr, '\0', sizeof(nladdr) );
	nladdr.nl_family = AF_NETLINK;
	if ( bind( h->fd, (struct sockaddr *) &nladdr, sizeof(nladdr) ) < 0 )
	{
close_and_out:
		__netlink_close( h );
out:
		return(-1);
	}


	/* Determine the ID the kernel assigned for this netlink connection.
	 * It is not necessarily the PID if there is more than one socket
	 * open.  */
	socklen_t addr_len = sizeof(nladdr);
	if ( getsockname( h->fd, (struct sockaddr *) &nladdr, &addr_len ) < 0 )
		goto close_and_out;
	h->pid = nladdr.nl_pid;
	return(0);
}


/* We know the number of RTM_NEWLINK entries, so we reserve the first
 # of entries for this type. All RTM_NEWADDR entries have an index
 # pointer to the RTM_NEWLINK entry.  To find the entry, create
 # a table to map kernel index entries to our index numbers.
 # Since we get at first all RTM_NEWLINK entries, it can never happen
 # that a RTM_NEWADDR index is not known to this map.  */
static int
map_newlink( int idx, struct ifaddrs_storage *ifas, int *map, int max )
{
	int i;

	for ( i = 0; i < max; i++ )
	{
		if ( map[i] == -1 )
		{
			map[i] = idx;
			if ( i > 0 )
				ifas[i - 1].ifa.ifa_next = &ifas[i].ifa;
			return(i);
		}    else if ( map[i] == idx )
			return(i);
	}


	/* This should never be reached. If this will be reached, we have
	 * a very big problem.  */
	abort();
}


/* Create a linked list of `struct ifaddrs' structures, one for each
 * network interface on the host machine.  If successful, store the
 * list in *IFAP and return 0.  On errors, return -1 and set `errno'.  */
int
getifaddrs( struct ifaddrs **ifap )
{
	struct netlink_handle	nh = { 0, 0, 0, NULL, NULL };
	struct netlink_res	*nlp;
	struct ifaddrs_storage	*ifas;
	unsigned int		i, newlink, newaddr, newaddr_idx;
	int			*map_newlink_data;
	size_t			ifa_data_size = 0;      /* Size to allocate for all ifa_data.  */
	char			*ifa_data_ptr;          /* Pointer to the unused part of memory for
	                                                 * ifa_data.  */
	int result = 0;

	if ( ifap )
		*ifap = NULL;

	if ( __netlink_open( &nh ) < 0 )
	{
		return(-1);
	}


	/* Tell the kernel that we wish to get a list of all
	 * active interfaces, collect all data for every interface.  */
	if ( __netlink_request( &nh, RTM_GETLINK ) < 0 )
	{
		printf( "%s %d\n", __FUNCTION__, __LINE__ );
		result = -1;
		goto exit_free;
	}


	/* Now ask the kernel for all addresses which are assigned
	 * to an interface and collect all data for every interface.
	 * Since we store the addresses after the interfaces in the
	 * list, we will later always find the interface before the
	 * corresponding addresses.  */
	++nh.seq;
	if ( __netlink_request( &nh, RTM_GETADDR ) < 0 )
	{
		printf( "%s %d\n", __FUNCTION__, __LINE__ );
		result = -1;
		goto exit_free;
	}


	/* Count all RTM_NEWLINK and RTM_NEWADDR entries to allocate
	 * enough memory.  */
	newlink = newaddr = 0;
	for ( nlp = nh.nlm_list; nlp; nlp = nlp->next )
	{
		struct nlmsghdr *nlh;
		size_t		size = nlp->size;

		if ( nlp->nlh == NULL )
			continue;


		/* Walk through all entries we got from the kernel and look, which
		 * message type they contain.  */
		for ( nlh = nlp->nlh; NLMSG_OK( nlh, size ); nlh = NLMSG_NEXT( nlh, size ) )
		{
			/* Check if the message is what we want.  */
			if ( (pid_t) nlh->nlmsg_pid != nh.pid || nlh->nlmsg_seq != nlp->seq )
				continue;

			if ( nlh->nlmsg_type == NLMSG_DONE )
				break;  /* ok */

			if ( nlh->nlmsg_type == RTM_NEWLINK )
			{
				/* A RTM_NEWLINK message can have IFLA_STATS data. We need to
				 * know the size before creating the list to allocate enough
				 * memory.  */
				struct ifinfomsg	*ifim	= (struct ifinfomsg *) NLMSG_DATA( nlh );
				struct rtattr		*rta	= IFLA_RTA( ifim );
				size_t			rtasize = IFLA_PAYLOAD( nlh );

				while ( RTA_OK( rta, rtasize ) )
				{
					size_t rta_payload = RTA_PAYLOAD( rta );

					if ( rta->rta_type == IFLA_STATS )
					{
						ifa_data_size += rta_payload;
						break;
					}else
						rta = RTA_NEXT( rta, rtasize );
				}
				++newlink;
			}else if ( nlh->nlmsg_type == RTM_NEWADDR )
				++newaddr;
		}
	}

	/* Return if no interface is up.  */
	if ( (newlink + newaddr) == 0 )
		goto exit_free;


	/* Allocate memory for all entries we have and initialize next
	 * pointer.  */
	ifas = calloc( 1, (newlink + newaddr) * sizeof(ifas[0]) + ifa_data_size );
	if ( ifas == NULL )
	{
		result = -1;
		printf( "%s %d\n", __FUNCTION__, __LINE__ );
		goto exit_free;
	}

	/* Table for mapping kernel index to entry in our list.  */
	map_newlink_data = alloca( newlink * sizeof(int) );
	memset( map_newlink_data, '\xff', newlink * sizeof(int) );

	ifa_data_ptr	= (char *) &ifas[newlink + newaddr];
	newaddr_idx	= 0; /* Counter for newaddr index.  */

	/* Walk through the list of data we got from the kernel.  */
	for ( nlp = nh.nlm_list; nlp; nlp = nlp->next )
	{
		struct nlmsghdr *nlh;
		size_t		size = nlp->size;

		if ( nlp->nlh == NULL )
			continue;


		/* Walk through one message and look at the type: If it is our
		 * message, we need RTM_NEWLINK/RTM_NEWADDR and stop if we reach
		 * the end or we find the end marker (in this case we ignore the
		 * following data.  */
		for ( nlh = nlp->nlh; NLMSG_OK( nlh, size ); nlh = NLMSG_NEXT( nlh, size ) )
		{
			int ifa_index = 0;

			/* Check if the message is the one we want */
			if ( (pid_t) nlh->nlmsg_pid != nh.pid || nlh->nlmsg_seq != nlp->seq )
				continue;

			if ( nlh->nlmsg_type == NLMSG_DONE )
				break;  /* ok */

			if ( nlh->nlmsg_type == RTM_NEWLINK )
			{
				/* We found a new interface. Now extract everything from the
				 * interface data we got and need.  */
				struct ifinfomsg	*ifim	= (struct ifinfomsg *) NLMSG_DATA( nlh );
				struct rtattr		*rta	= IFLA_RTA( ifim );
				size_t			rtasize = IFLA_PAYLOAD( nlh );


				/* Interfaces are stored in the first "newlink" entries
				 * of our list, starting in the order as we got from the
				 * kernel.  */
				ifa_index = map_newlink( ifim->ifi_index - 1, ifas,
							 map_newlink_data, newlink );
				ifas[ifa_index].ifa.ifa_flags = ifim->ifi_flags;

				while ( RTA_OK( rta, rtasize ) )
				{
					char	*rta_data	= RTA_DATA( rta );
					size_t	rta_payload	= RTA_PAYLOAD( rta );

					switch ( rta->rta_type )
					{
					case IFLA_ADDRESS:
						if ( rta_payload <= sizeof(ifas[ifa_index].addr) )
						{
							ifas[ifa_index].addr.sl.sll_family = AF_PACKET;
							memcpy( ifas[ifa_index].addr.sl.sll_addr,
								(char *) rta_data, rta_payload );
							ifas[ifa_index].addr.sl.sll_halen = rta_payload;
							ifas[ifa_index].addr.sl.sll_ifindex
												= ifim->ifi_index;
							ifas[ifa_index].addr.sl.sll_hatype	= ifim->ifi_type;

							ifas[ifa_index].ifa.ifa_addr
								= &ifas[ifa_index].addr.sa;
						}
						break;

					case IFLA_BROADCAST:
						if ( rta_payload <= sizeof(ifas[ifa_index].broadaddr) )
						{
							ifas[ifa_index].broadaddr.sl.sll_family = AF_PACKET;
							memcpy( ifas[ifa_index].broadaddr.sl.sll_addr,
								(char *) rta_data, rta_payload );
							ifas[ifa_index].broadaddr.sl.sll_halen = rta_payload;
							ifas[ifa_index].broadaddr.sl.sll_ifindex
								= ifim->ifi_index;
							ifas[ifa_index].broadaddr.sl.sll_hatype
								= ifim->ifi_type;

							ifas[ifa_index].ifa.ifa_broadaddr
								= &ifas[ifa_index].broadaddr.sa;
						}
						break;

					case IFLA_IFNAME: /* Name of Interface */
						if ( (rta_payload + 1) <= sizeof(ifas[ifa_index].name) )
						{
							ifas[ifa_index].ifa.ifa_name = ifas[ifa_index].name;
							*(char *) mempcpy( ifas[ifa_index].name, rta_data,
									   rta_payload ) = '\0';
						}
						break;

					case IFLA_STATS: /* Statistics of Interface */
						ifas[ifa_index].ifa.ifa_data	= ifa_data_ptr;
						ifa_data_ptr			+= rta_payload;
						memcpy( ifas[ifa_index].ifa.ifa_data, rta_data,
							rta_payload );
						break;

					case IFLA_UNSPEC:
						break;
					case IFLA_MTU:
						break;
					case IFLA_LINK:
						break;
					case IFLA_QDISC:
						break;
					default:
						break;
					}

					rta = RTA_NEXT( rta, rtasize );
				}
			}else if ( nlh->nlmsg_type == RTM_NEWADDR )
			{
				struct ifaddrmsg	*ifam	= (struct ifaddrmsg *) NLMSG_DATA( nlh );
				struct rtattr		*rta	= IFA_RTA( ifam );
				size_t			rtasize = IFA_PAYLOAD( nlh );


				/* New Addresses are stored in the order we got them from
				 * the kernel after the interfaces. Theoretically it is possible
				 * that we have holes in the interface part of the list,
				 * but we always have already the interface for this address.  */
				ifa_index = newlink + newaddr_idx;
				ifas[ifa_index].ifa.ifa_flags
					= ifas[map_newlink( ifam->ifa_index - 1, ifas,
							    map_newlink_data, newlink )].ifa.ifa_flags;
				if ( ifa_index > 0 )
					ifas[ifa_index - 1].ifa.ifa_next = &ifas[ifa_index].ifa;
				++newaddr_idx;

				while ( RTA_OK( rta, rtasize ) )
				{
					char	*rta_data	= RTA_DATA( rta );
					size_t	rta_payload	= RTA_PAYLOAD( rta );

					switch ( rta->rta_type )
					{
					case IFA_ADDRESS:
					{
						struct sockaddr *sa;

						if ( ifas[ifa_index].ifa.ifa_addr != NULL )
						{
							/* In a point-to-poing network IFA_ADDRESS
							 * contains the destination address, local
							 * address is supplied in IFA_LOCAL attribute.
							 * destination address and broadcast address
							 * are stored in an union, so it doesn't matter
							 * which name we use.  */
							ifas[ifa_index].ifa.ifa_broadaddr
								= &ifas[ifa_index].broadaddr.sa;
							sa	= &ifas[ifa_index].broadaddr.sa;
						}else    {
							ifas[ifa_index].ifa.ifa_addr
								= &ifas[ifa_index].addr.sa;
							sa	= &ifas[ifa_index].addr.sa;
						}

						sa->sa_family = ifam->ifa_family;

						switch ( ifam->ifa_family )
						{
						case AF_INET:
							/* Size must match that of an address for IPv4.  */
							if ( rta_payload == 4 )
								memcpy( &( (struct sockaddr_in *) sa)->sin_addr,
									rta_data, rta_payload );
							break;

#ifdef __UCLIBC_HAS_IPV6__
						case AF_INET6:
							/* Size must match that of an address for IPv6.  */
							if ( rta_payload == 16 )
							{
								memcpy( &( (struct sockaddr_in6 *) sa)->sin6_addr,
									rta_data, rta_payload );
								if ( IN6_IS_ADDR_LINKLOCAL( rta_data )
								     || IN6_IS_ADDR_MC_LINKLOCAL( rta_data ) )
									( (struct sockaddr_in6 *) sa)->sin6_scope_id
										= ifam->ifa_index;
							}
							break;
#endif

						default:
							if ( rta_payload <= sizeof(ifas[ifa_index].addr) )
								memcpy( sa->sa_data, rta_data, rta_payload );
							break;
						}
					}
					break;

					case IFA_LOCAL:
						if ( ifas[ifa_index].ifa.ifa_addr != NULL )
						{
							/* If ifa_addr is set and we get IFA_LOCAL,
							 * assume we have a point-to-point network.
							 * Move address to correct field.  */
							ifas[ifa_index].broadaddr = ifas[ifa_index].addr;
							ifas[ifa_index].ifa.ifa_broadaddr
								= &ifas[ifa_index].broadaddr.sa;
							memset( &ifas[ifa_index].addr, '\0',
								sizeof(ifas[ifa_index].addr) );
						}

						ifas[ifa_index].ifa.ifa_addr = &ifas[ifa_index].addr.sa;
						ifas[ifa_index].ifa.ifa_addr->sa_family
							= ifam->ifa_family;

						switch ( ifam->ifa_family )
						{
						case AF_INET:
							/* Size must match that of an address for IPv4.  */
							if ( rta_payload == 4 )
								memcpy( &ifas[ifa_index].addr.s4.sin_addr,
									rta_data, rta_payload );
							break;

#ifdef __UCLIBC_HAS_IPV6__
						case AF_INET6:
							/* Size must match that of an address for IPv6.  */
							if ( rta_payload == 16 )
							{
								memcpy( &ifas[ifa_index].addr.s6.sin6_addr,
									rta_data, rta_payload );
								if ( IN6_IS_ADDR_LINKLOCAL( rta_data )
								     || IN6_IS_ADDR_MC_LINKLOCAL( rta_data ) )
									ifas[ifa_index].addr.s6.sin6_scope_id =
										ifam->ifa_index;
							}
							break;
#endif

						default:
							if ( rta_payload <= sizeof(ifas[ifa_index].addr) )
								memcpy( ifas[ifa_index].addr.sa.sa_data,
									rta_data, rta_payload );
							break;
						}
						break;

					case IFA_BROADCAST:
						/* We get IFA_BROADCAST, so IFA_LOCAL was too much.  */
						if ( ifas[ifa_index].ifa.ifa_broadaddr != NULL )
							memset( &ifas[ifa_index].broadaddr, '\0',
								sizeof(ifas[ifa_index].broadaddr) );

						ifas[ifa_index].ifa.ifa_broadaddr
							= &ifas[ifa_index].broadaddr.sa;
						ifas[ifa_index].ifa.ifa_broadaddr->sa_family
							= ifam->ifa_family;

						switch ( ifam->ifa_family )
						{
						case AF_INET:
							/* Size must match that of an address for IPv4.  */
							if ( rta_payload == 4 )
								memcpy( &ifas[ifa_index].broadaddr.s4.sin_addr,
									rta_data, rta_payload );
							break;

#ifdef __UCLIBC_HAS_IPV6__
						case AF_INET6:
							/* Size must match that of an address for IPv6.  */
							if ( rta_payload == 16 )
							{
								memcpy( &ifas[ifa_index].broadaddr.s6.sin6_addr,
									rta_data, rta_payload );
								if ( IN6_IS_ADDR_LINKLOCAL( rta_data )
								     || IN6_IS_ADDR_MC_LINKLOCAL( rta_data ) )
									ifas[ifa_index].broadaddr.s6.sin6_scope_id
										= ifam->ifa_index;
							}
							break;
#endif

						default:
							if ( rta_payload <= sizeof(ifas[ifa_index].addr) )
								memcpy( &ifas[ifa_index].broadaddr.sa.sa_data,
									rta_data, rta_payload );
							break;
						}
						break;

					case IFA_LABEL:
						if ( rta_payload + 1 <= sizeof(ifas[ifa_index].name) )
						{
							ifas[ifa_index].ifa.ifa_name = ifas[ifa_index].name;
							*(char *) mempcpy( ifas[ifa_index].name, rta_data,
									   rta_payload ) = '\0';
						}    else
							abort();
						break;

					case IFA_UNSPEC:
						break;
					case IFA_CACHEINFO:
						break;
					default:
						break;
					}

					rta = RTA_NEXT( rta, rtasize );
				}


				/* If we didn't get the interface name with the
				 * address, use the name from the interface entry.  */
				if ( ifas[ifa_index].ifa.ifa_name == NULL )
					ifas[ifa_index].ifa.ifa_name
						= ifas[map_newlink( ifam->ifa_index - 1, ifas,
								    map_newlink_data, newlink )].ifa.ifa_name;

				/* Calculate the netmask.  */
				if ( ifas[ifa_index].ifa.ifa_addr
				     && ifas[ifa_index].ifa.ifa_addr->sa_family != AF_UNSPEC
				     && ifas[ifa_index].ifa.ifa_addr->sa_family != AF_PACKET )
				{
					uint32_t	max_prefixlen	= 0;
					char		*cp		= NULL;

					ifas[ifa_index].ifa.ifa_netmask
						= &ifas[ifa_index].netmask.sa;

					switch ( ifas[ifa_index].ifa.ifa_addr->sa_family )
					{
					case AF_INET:
						cp		= (char *) &ifas[ifa_index].netmask.s4.sin_addr;
						max_prefixlen	= 32;
						break;

#ifdef __UCLIBC_HAS_IPV6__
					case AF_INET6:
						cp		= (char *) &ifas[ifa_index].netmask.s6.sin6_addr;
						max_prefixlen	= 128;
						break;
#endif
					}

					ifas[ifa_index].ifa.ifa_netmask->sa_family
						= ifas[ifa_index].ifa.ifa_addr->sa_family;

					if ( cp != NULL )
					{
						char		c;
						unsigned int	preflen;

						if ( (max_prefixlen > 0) &&
						     (ifam->ifa_prefixlen > max_prefixlen) )
							preflen = max_prefixlen;
						else
							preflen = ifam->ifa_prefixlen;

						for ( i = 0; i < (preflen / 8); i++ )
							*cp++ = 0xff;
						c	= 0xff;
						c	<<= (8 - (preflen % 8) );
						*cp	= c;
					}
				}
			}
		}
	}

	assert( ifa_data_ptr <= (char *) &ifas[newlink + newaddr] + ifa_data_size );

	if ( newaddr_idx > 0 )
	{
		for ( i = 0; i < newlink; ++i )
			if ( map_newlink_data[i] == -1 )
			{
				/* We have fewer links then we anticipated.  Adjust the
				 * forward pointer to the first address entry.  */
				ifas[i - 1].ifa.ifa_next = &ifas[newlink].ifa;
			}

		if ( i == 0 && newlink > 0 )
			/* No valid link, but we allocated memory.  We have to
			 * populate the first entry.  */
			memmove( ifas, &ifas[newlink], sizeof(struct ifaddrs_storage) );
	}

	if ( ifap != NULL )
		*ifap = &ifas[0].ifa;

exit_free:
	__netlink_free_handle( &nh );
	__netlink_close( &nh );

	return(result);
}


void
freeifaddrs( struct ifaddrs *ifa )
{
	free( ifa );
}

#endif
int main( int argc, char* argv[] )
{
	struct ifaddrs	*ifc, *ifc1;
	char		ip[64] = {};
	char		nm[64] = {};
	int		ret = 0;
	if ( 0 != (ret = getifaddrs( &ifc ) ) )
	{
		printf( "ret=%d\n", ret );
		perror( "getifaddrs" );
		return(-1);
	}
	ifc1 = ifc;

	printf( "iface\tIP address\tNetmask\n" );
	for (; NULL != ifc; ifc = (*ifc).ifa_next )
	{
		printf( "%s", (*ifc).ifa_name );
		if ( NULL != (*ifc).ifa_addr )
		{
			inet_ntop( AF_INET, &( ( (struct sockaddr_in *) ( (*ifc).ifa_addr) )->sin_addr), ip, 64 );
			printf( "\t%s", ip );
		}else{
			printf( "\t\t" );
		}
		if ( NULL != (*ifc).ifa_netmask )
		{
			inet_ntop( AF_INET, &( ( (struct sockaddr_in *) ( (*ifc).ifa_netmask) )->sin_addr), nm, 64 );
			printf( "\t%s", nm );
		}else{
			printf( "\t\t" );
		}
		printf( "\n" );
	}

	/* freeifaddrs(ifc); */
	freeifaddrs( ifc1 );
	return(0);
}



```

## 问题分析
    通过查看代码发现主要是向netlink发送查询报文（__netlink_request RTM_GETLINK）后，内核返回netlink消息err EINTR 和EBUSY。跟踪内核netlink对应的代码。
    对应代码在net/core/rtnetlink.c
    
```
void __init rtnetlink_init(void)
{
	if (register_pernet_subsys(&rtnetlink_net_ops))
		panic("rtnetlink_init: cannot initialize rtnetlink\n");

	register_netdevice_notifier(&rtnetlink_dev_notifier);

	rtnl_register(PF_UNSPEC, RTM_GETLINK, rtnl_getlink,
		      rtnl_dump_ifinfo, rtnl_calcit);
    .......
}
```
rtnetlink初始化入口函数中注册了对应input函数，

```

static int __net_init rtnetlink_net_init(struct net *net)
{
	struct sock *sk;
	struct netlink_kernel_cfg cfg = {
		.groups		= RTNLGRP_MAX,
		.input		= rtnetlink_rcv,
		.cb_mutex	= &rtnl_mutex,
		.flags		= NL_CFG_F_NONROOT_RECV,
	};

	sk = netlink_kernel_create(net, NETLINK_ROUTE, &cfg);
	if (!sk)
		return -ENOMEM;
	net->rtnl = sk;
	return 0;
}

```
也就是当应用层发送NETLINK_ROUTE消息时对应的接收函数 rtnetlink_rcv收到响应数据，然后再调用rtnetlink_rcv_msg，

```

static int rtnetlink_rcv_msg(struct sk_buff *skb, struct nlmsghdr *nlh)
{
	struct net *net = sock_net(skb->sk);
	rtnl_doit_func doit;
	int sz_idx, kind;
	int family;
	int type;
	int err;

	type = nlh->nlmsg_type;
	if (type > RTM_MAX)
		return -EOPNOTSUPP;

	type -= RTM_BASE;
//请求为RTM_GETLINK type=RTM_GETLINK-RTM_BASE=2
	/* All the messages must have at least 1 byte length */
	if (nlmsg_len(nlh) < sizeof(struct rtgenmsg))
		return 0;

	family = ((struct rtgenmsg *)nlmsg_data(nlh))->rtgen_family;
	sz_idx = type>>2;
	kind = type&3;

	if (kind != 2 && !netlink_net_capable(skb, CAP_NET_ADMIN))
		return -EPERM;
//NLM_F_DUMP定义为NLM_F_ROOT | NLM_F_MATCH ，所以只要有设置这两个标识，就会进入下面的if语句
	if (kind == 2 && nlh->nlmsg_flags&NLM_F_DUMP) {
		struct sock *rtnl;
		rtnl_dumpit_func dumpit;
		rtnl_calcit_func calcit;
		u16 min_dump_alloc = 0;

		dumpit = rtnl_get_dumpit(family, type);
//通过family和type查找rtnl_msg_handlers 也就是刚刚注册的rtnl_dump_ifinfo函数
		if (dumpit == NULL)
			return -EOPNOTSUPP;
		calcit = rtnl_get_calcit(family, type);
		if (calcit)
			min_dump_alloc = calcit(skb, nlh);

		__rtnl_unlock();
		rtnl = net->rtnl;
		{
			struct netlink_dump_control c = {
				.dump		= dumpit,
				.min_dump_alloc	= min_dump_alloc,
			};
			err = netlink_dump_start(rtnl, skb, nlh, &c);
			//赋值对应的netlink_dump_control结构体后，调用dump函数
		}
		rtnl_lock();
		return err;
	}

	doit = rtnl_get_doit(family, type);
	if (doit == NULL)
		return -EOPNOTSUPP;

	return doit(skb, nlh);
}
```
doit函数 doit(skb, nlh)未执行到就返回了，直接netlink_dump_start后返回，netlink_dump_start函数在net/netlink/af_netlink.c中定义实现，如下

```

int __netlink_dump_start(struct sock *ssk, struct sk_buff *skb,
			 const struct nlmsghdr *nlh,
			 struct netlink_dump_control *control)
{
	struct netlink_callback *cb;
	struct sock *sk;
	struct netlink_sock *nlk;
	int ret;

	cb = kzalloc(sizeof(*cb), GFP_KERNEL);
	if (cb == NULL)
		return -ENOBUFS;

	/* Memory mapped dump requests need to be copied to avoid looping
	 * on the pending state in netlink_mmap_sendmsg() while the CB hold
	 * a reference to the skb.
	 */
	if (netlink_skb_is_mmaped(skb)) {
		skb = skb_copy(skb, GFP_KERNEL);
		if (skb == NULL) {
			kfree(cb);
			return -ENOBUFS;
		}
	} else
		atomic_inc(&skb->users);
//给netlink_callback赋值为刚刚传入参数
**	cb->dump = control->dump;**
	cb->done = control->done;
	cb->nlh = nlh;
	cb->data = control->data;
	cb->module = control->module;
	cb->min_dump_alloc = control->min_dump_alloc;
	cb->skb = skb;
//找到对应的sk
	sk = netlink_lookup(sock_net(ssk), ssk->sk_protocol, NETLINK_CB(skb).portid);
	if (sk == NULL) {
		netlink_destroy_callback(cb);
		return -ECONNREFUSED;
	}
	nlk = nlk_sk(sk);

	mutex_lock(nlk->cb_mutex);
	/* A dump is in progress... */
	//如果判断到有一个dump已经在了，就直接返回busy，也就是应用层收到的EBUSY
	if (nlk->cb) {
		mutex_unlock(nlk->cb_mutex);
		netlink_destroy_callback(cb);
		ret = -EBUSY;
		goto out;
	}
	/* add reference of module which cb->dump belongs to */
	if (!try_module_get(cb->module)) {
		mutex_unlock(nlk->cb_mutex);
		netlink_destroy_callback(cb);
		ret = -EPROTONOSUPPORT;
		goto out;
	}

	nlk->cb = cb;
	mutex_unlock(nlk->cb_mutex);
//调用关键函数
	ret = netlink_dump(sk);
out:
	sock_put(sk);

	if (ret)
		return ret;

	/* We successfully started a dump, by returning -EINTR we
	 * signal not to send ACK even if it was requested.
	 */
	 //第一次返回用户态的值在这里EINTR
	return -EINTR;
}
EXPORT_SYMBOL(__netlink_dump_start);
```
netlink_dump函数中读取所有接口信息并封装为数据包，然后应用层recvmsg接收

```

static int netlink_dump(struct sock *sk)
{
	struct netlink_sock *nlk = nlk_sk(sk);
	struct netlink_callback *cb;
	struct sk_buff *skb = NULL;
	struct nlmsghdr *nlh;
	int len, err = -ENOBUFS;
	int alloc_size;

	mutex_lock(nlk->cb_mutex);

	cb = nlk->cb;
	if (cb == NULL) {
		err = -EINVAL;
		goto errout_skb;
	}

	alloc_size = max_t(int, cb->min_dump_alloc, NLMSG_GOODSIZE);

	if (!netlink_rx_is_mmaped(sk) &&
	    atomic_read(&sk->sk_rmem_alloc) >= sk->sk_rcvbuf)
		goto errout_skb;
	skb = netlink_alloc_skb(sk, alloc_size, nlk->portid, GFP_KERNEL);
	if (!skb)
		goto errout_skb;
	netlink_skb_set_owner_r(skb, sk);
//这里调用最开始注册的dump函数rtnl_dump_ifinfo
	len = cb->dump(skb, cb);

	if (len > 0) {
		mutex_unlock(nlk->cb_mutex);

		if (sk_filter(sk, skb))
			kfree_skb(skb);
		else
			__netlink_sendskb(sk, skb);
		return 0;
	}
//上层应用能够收到NLMSG_DONE的消息条件就是len<=0,目前是应用层完全没有收到NLMSG_DONE，也就是每次从上面if语句返回了
	nlh = nlmsg_put_answer(skb, cb, NLMSG_DONE, sizeof(len), NLM_F_MULTI);
	if (!nlh)
		goto errout_skb;

	nl_dump_check_consistent(cb, nlh);

	memcpy(nlmsg_data(nlh), &len, sizeof(len));

	if (sk_filter(sk, skb))
		kfree_skb(skb);
	else
		__netlink_sendskb(sk, skb);

	if (cb->done)
		cb->done(cb);
	nlk->cb = NULL;
	mutex_unlock(nlk->cb_mutex);

	module_put(cb->module);
	netlink_consume_callback(cb);
	return 0;

errout_skb:
	mutex_unlock(nlk->cb_mutex);
	kfree_skb(skb);
	return err;
}
```
进一步查看	len = cb->dump(skb, cb);也就是rtnl_dump_ifinfo，这个函数返回值len需要为小于等于0 的条件就是没有进入for循环调用rtnl_fill_ifinfo来改变skb长度。
该函数有一个关键点在于结束条件，就是通过每次进入函数时，记录上次读取的进度	s_h = cb->args[0];
然后退出时保存	cb->args[0] = h;这样就可以当遍历完NETDEV_HASHENTRIES后，函数返回值不大于0

```

static int rtnl_dump_ifinfo(struct sk_buff *skb, struct netlink_callback *cb)
{
	struct net *net = sock_net(skb->sk);
	int h, s_h;
	int idx = 0, s_idx;
	struct net_device *dev;
	struct hlist_head *head;
	struct nlattr *tb[IFLA_MAX+1];
	u32 ext_filter_mask = 0;
	int err;
	int hdrlen;

	s_h = cb->args[0];
	//上次读取的进度
	s_idx = cb->args[1];

	rcu_read_lock();
	cb->seq = net->dev_base_seq;

	/* A hack to preserve kernel<->userspace interface.
	 * The correct header is ifinfomsg. It is consistent with rtnl_getlink.
	 * However, before Linux v3.9 the code here assumed rtgenmsg and that's
	 * what iproute2 < v3.9.0 used.
	 * We can detect the old iproute2. Even including the IFLA_EXT_MASK
	 * attribute, its netlink message is shorter than struct ifinfomsg.
	 */
	hdrlen = nlmsg_len(cb->nlh) < sizeof(struct ifinfomsg) ?
		 sizeof(struct rtgenmsg) : sizeof(struct ifinfomsg);

	if (nlmsg_parse(cb->nlh, hdrlen, tb, IFLA_MAX, ifla_policy) >= 0) {

		if (tb[IFLA_EXT_MASK])
			ext_filter_mask = nla_get_u32(tb[IFLA_EXT_MASK]);
	}

	for (h = s_h; h < NETDEV_HASHENTRIES; h++, s_idx = 0) {
		idx = 0;
		head = &net->dev_index_head[h];
		hlist_for_each_entry_rcu(dev, head, index_hlist) {
			if (idx < s_idx)
				goto cont;
			//将设备信息填充到skb中
			err = rtnl_fill_ifinfo(skb, dev, RTM_NEWLINK,
					       NETLINK_CB(cb->skb).portid,
					       cb->nlh->nlmsg_seq, 0,
					       NLM_F_MULTI,
					       ext_filter_mask);
			/* If we ran out of room on the first message,
			 * we're in trouble
			 */
			WARN_ON((err == -EMSGSIZE) && (skb->len == 0));

			if (err <= 0)
				goto out;

			nl_dump_check_consistent(cb, nlmsg_hdr(skb));
cont:
			idx++;
		}
	}
out:
	rcu_read_unlock();
	cb->args[1] = idx;
	cb->args[0] = h;
	//保存进度

	return skb->len;
}
```
通过以上代码分析，知道返回正确值NLMSG_DONE需要多次读取调用netlink_dump，直到所有接口信息完成就可以返回
查看代码，在rtnetlink_rcv_msg中开始netlink_dump_start开始获取接口信息调用一次netlink_dump，在netlink_recvmsg函数中如果nlk->cb没有干掉，就会去读取netlink_dump
nlk->cb是需要在netlink_dump中当cb->dump返回不大于0时才释放。也就是本次查询接口信息dump读完后释放

```
static int netlink_recvmsg(struct kiocb *kiocb, struct socket *sock,
			   struct msghdr *msg, size_t len,
			   int flags)
{
................
    	if (nlk->cb && atomic_read(&sk->sk_rmem_alloc) <= sk->sk_rcvbuf / 2) {
		ret = netlink_dump(sk);
		if (ret) {
			sk->sk_err = -ret;
			sk->sk_error_report(sk);
		}
	}
.............
}
```
netlink_recvmsg函数为netlink_ops中注册，也就是应用层调用系统调用recvmsg函数时调用
```
static const struct proto_ops netlink_ops = {
    ..........
    .sendmsg =	netlink_sendmsg,
	.recvmsg =	netlink_recvmsg,
	.mmap =		netlink_mmap,
	.sendpage =	sock_no_sendpage,
};
```

## 应用层代码分析
发起请求后通过while循环接收内核消息
__netlink_sendreq
```
int
__netlink_request( struct netlink_handle *h, int type )
{
	struct netlink_res	*nlm_next;
	struct netlink_res	**new_nlm_list;
	static volatile size_t	buf_size = 4096;
	char			*buf;
	struct sockaddr_nl	nladdr;
	struct nlmsghdr		*nlmh;
	ssize_t			read_len;
	bool			done		= false;
	bool			use_malloc	= false;
//发起netlink请求
	if ( __netlink_sendreq( h, type ) < 0 )
	{
		printf( "%s %d\n", __FUNCTION__, __LINE__ );
		return(-1);
	}
//分配接收缓存大小buf_size 这里为4096
	size_t this_buf_size = buf_size;
	if ( __libc_use_alloca( this_buf_size ) )
		buf = alloca( this_buf_size );
	else{
		buf = malloc( this_buf_size );
		if ( buf != NULL )
			use_malloc = true;
		else
			goto out_fail;
	}

	struct iovec iov = { buf, this_buf_size };

	if ( h->nlm_list != NULL )
		new_nlm_list = &h->end_ptr->next;
	else
		new_nlm_list = &h->nlm_list;

	while ( !done )
	{
		struct msghdr msg =
		{
			(void *) &nladdr, sizeof(nladdr),
			&iov,		  1,
			NULL,		  0,
			0
		};
	printf( "%s %d recv h->fd %d h->seq %d , h->pid %d type%d\n", __FUNCTION__, __LINE__, h->fd, h->seq, h->pid,type);
	    //循环接收内核消息到缓冲msg中
		read_len = TEMP_FAILURE_RETRY(recvmsg( h->fd, &msg, 0 ) );
		if ( read_len < 0 )
			goto out_fail;

		if ( nladdr.nl_pid != 0 )
			continue;
//这里是问题关键点，如果内核返回的消息中设置了MSG_TRUNC标识，就进入if语句，重新分配内存为原来的两倍
//__builtin_expect(condition,0)表示unlikely的意思，用于编译器优化 就是很少发生这样的状况
		if ( __builtin_expect( msg.msg_flags & MSG_TRUNC, 0 ) )
		{
			if ( this_buf_size >= SIZE_MAX / 2 )
				goto out_fail;

			nlm_next = *new_nlm_list;
			while ( nlm_next != NULL )
			{
				struct netlink_res *tmpptr;

				tmpptr = nlm_next->next;
				free( nlm_next );
				nlm_next = tmpptr;
			}
			*new_nlm_list = NULL;

			if ( __libc_use_alloca( 2 * this_buf_size ) )
				buf = extend_alloca( buf, this_buf_size, 2 * this_buf_size );
			else{
				this_buf_size *= 2;

				char *new_buf = realloc( use_malloc ? buf : NULL, this_buf_size );
				if ( new_buf == NULL )
					goto out_fail;
				new_buf = buf;

				use_malloc = true;
			}
			buf_size = this_buf_size;

			iov.iov_base	= buf;
			iov.iov_len	= this_buf_size;


			/* Increase sequence number, so that we can distinguish
			 * between old and new request messages.  */
			h->seq++;
//重新分配内存后我们重新发起一次请求，然后继续接受消息。根据应用层出错信息，每次都会进入下面这里，但是进入后，就返回EBUSY,也就是上次的请求没有完成。
			if ( __netlink_sendreq( h, type ) < 0 )
			{
				printf( "%s %d h->seq, h->pid\n", __FUNCTION__, __LINE__, h->seq, h->pid );
				goto out_fail;
			}

			continue;
		}

		size_t	count		= 0;
		size_t	remaining_len	= read_len;
		for ( nlmh = (struct nlmsghdr *) buf;
		      NLMSG_OK( nlmh, remaining_len );
		      nlmh = (struct nlmsghdr *) NLMSG_NEXT( nlmh, remaining_len ) )
		{
			printf( "%s %d recv h->fd %d h->seq %d ,nlmh->nlmsg_seq %d ,nlmh->nlmsg_pid %d, h->pid %d type%d\n", __FUNCTION__, __LINE__, h->fd, h->seq,nlmh->nlmsg_seq,nlmh->nlmsg_pid , h->pid,type);
			if ( (pid_t) nlmh->nlmsg_pid != h->pid
			     || nlmh->nlmsg_seq != h->seq )
				continue;
			++count;
			if ( nlmh->nlmsg_type == NLMSG_DONE )
			{
				/* We found the end, leave the loop.  */
				done = true;
				break;
			}
			if ( nlmh->nlmsg_type == NLMSG_ERROR )
			{
				struct nlmsgerr *nlerr = (struct nlmsgerr *) NLMSG_DATA( nlmh );
				if ( nlmh->nlmsg_len < NLMSG_LENGTH( sizeof(struct nlmsgerr) ) )
					errno = EIO;
				else
					errno = -nlerr->error;
				//两次都从这里退出，第二次是EBUSY错误
				printf( "%s %d  errno%d\n", __FUNCTION__, __LINE__,errno);
				goto out_fail;
			}
		}


		/* If there was nothing with the expected nlmsg_pid and nlmsg_seq,
		 * there is no point to record it.  */
		if ( count == 0 )
			continue;

		nlm_next = (struct netlink_res *) malloc( sizeof(struct netlink_res)
							  + read_len );
		if ( nlm_next == NULL )
			goto out_fail;
		nlm_next->next	= NULL;
		nlm_next->nlh	= memcpy( nlm_next + 1, buf, read_len );
		nlm_next->size	= read_len;
		nlm_next->seq	= h->seq;
		if ( h->nlm_list == NULL )
			h->nlm_list = nlm_next;
		else
			h->end_ptr->next = nlm_next;
		h->end_ptr = nlm_next;
	}

	if ( use_malloc )
		free( buf );
	return(0);

out_fail:
	if ( use_malloc )
		free( buf );
	printf( "%s %d\n", __FUNCTION__, __LINE__ );
	return(-1);
}


```
通过上面分析，知道应用层第一次请求没有接收完，或者无法接收，导致发起第二次，以2*this_buf_size接收，但是第二次又因为第一次请求还在处理，返回EBUSY.根本原因就是为什么第一次没有接收。MSG_TRUNC标识表示内核返回信息被截断了，需要继续接收，但是在这个libc函数中没有继续处理，或者就是直接第一次将数据接收完全。

## 总结
    因libc库的接收缓冲只有4096大小，但是内核接口较多（我这里有32个接口，大概15K的样子）导致recvmsg无法完全收取内核消息。而libc库对于该情况直接发起第二次请求，内核中不知道第一次请求是否完成，所以没有销毁第一次请求的netlink_callback，导致返回EBUSY.
    解决可以增大接收缓冲为16384，或者减少接口


  