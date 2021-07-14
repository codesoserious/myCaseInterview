# myCaseInterview
手把手带你入门 Spring Security！:https://www.cnblogs.com/lenve/p/11242055.html
SSO单点登录：https://www.cnblogs.com/ZhuChangwu/p/11997499.html
---------

import org.apache.commons.lang3.RandomUtils;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.SystemUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.net.Inet4Address;
import java.net.UnknownHostException;
import java.util.Locale;

/**
 * 唯一单号实现
 *
 * @author zwx990303
 * @since @since 2021/7/7
 */
public class UniqueIdUtil {
    private static final Logger LOG = LoggerFactory.getLogger(UniqueIdUtil.class);

    /**
     * 起始的时间戳从(2021/7/6)
     */
    private static final long START_TIMESTAMP = 1625629664226L;

    /**
     * 每一部分占用的位数(序列号占用的位数)
     */
    private final static long SEQUENCE_BIT = 12;

    /**
     * 机器标识占用的位数
     */
    private final static long MACHINE_BIT = 5;

    /**
     * 数据中心占用的位数
     */
    private final static long DATACENTER_BIT = 5;

    /**
     * 每一部分的最大值
     */
    private final static long MAX_DATACENTER_NUM = ~(-1L << DATACENTER_BIT);

    private final static long MAX_MACHINE_NUM = ~(-1L << MACHINE_BIT);

    private final static long MAX_SEQUENCE = ~(-1L << SEQUENCE_BIT);

    /**
     * 每一部分向左的位移
     */
    private final static long MACHINE_LEFT = SEQUENCE_BIT;

    private final static long DATACENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;

    private final static long TIMESTMP_LEFT = DATACENTER_LEFT + DATACENTER_BIT;

    private static UniqueIdUtil idUtil;

    static {
        LOG.info("init UniqueIdUtil start");
        idUtil = new UniqueIdUtil(getDataCenterId(), getMachineId());
    }

    /**
     * 数据中心
     */
    private final long datacenterId;

    /**
     * 机器标识
     */
    private final long machineId;

    /**
     * 序列号
     */
    private long sequence = 0L;

    /**
     * 上一次时间戳
     */
    private long lastStmp = -1L;

    /**
     * 唯一id有参构造(数据中心id,机器id)
     *
     * @param datacenterId 数据中心id
     * @param machineId 机器Id
     */
    public UniqueIdUtil(long datacenterId, long machineId) {
        if (datacenterId > MAX_DATACENTER_NUM || datacenterId < 0) {
            throw new IllegalArgumentException("datacenterId can't be greater than MAX_DATACENTER_NUM or less than 0");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("machineId can't be greater than MAX_MACHINE_NUM or less than 0");
        }
        this.datacenterId = datacenterId;
        this.machineId = machineId;
    }

    /**
     * 获取数据中心id
     *
     * @return 数据中心
     */
    private static Long getDataCenterId() {
        int[] ints = StringUtils.toCodePoints(SystemUtils.getHostName());
        int sums = 0;
        for (int i : ints) {
            sums += i;
        }
        return (long) (sums % 32);
    }

    /**
     * 产生下一个ID
     *
     * @return 下一个唯一id
     */
    public synchronized Long nextId() {
        long currTimestamp = getNewstmp();
        if (currTimestamp < lastStmp) {
            LOG.error(String.format(Locale.ROOT, "Clock moved backwards.  Refusing to generate id for %d milliseconds ",
                lastStmp));
            return null;
        } else if (currTimestamp == lastStmp) {
            // 相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            // 同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                currTimestamp = getNextMill();
            }
        } else {
            // 不同毫秒内，序列号置为0
            sequence = 0L;
        }
        lastStmp = currTimestamp;
        // 1、时间戳部分;2、数据中心部分;3、机器标识部分;4、序列号部分
        return (currTimestamp - START_TIMESTAMP) << TIMESTMP_LEFT | datacenterId << DATACENTER_LEFT
            | machineId << MACHINE_LEFT | sequence;
    }

    /**
     * (阻塞到下一毫秒，并获取最新时间戳)
     *
     * @return 下一个毫秒数
     */
    private long getNextMill() {
        long mill = getNewstmp();
        while (mill <= lastStmp) {
            mill = getNewstmp();
        }
        return mill;
    }

    /**
     * 获取新的时间串
     *
     * @return newStmp
     */
    private static long getNewstmp() {
        return System.currentTimeMillis();
    }

    /**
     * 机器标识id
     *
     * @return 机器标识id
     */
    private static Long getMachineId() {
        try {
            String hostAddress = Inet4Address.getLocalHost().getHostAddress();
            int[] ints = StringUtils.toCodePoints(hostAddress);
            int sums = 0;
            for (int b : ints) {
                sums += b;
            }
            return (long) (sums % 32);
        } catch (UnknownHostException unknown) {
            LOG.error("UnknownHostException ", unknown);
            // 如果获取失败，则使用随机数备用
            return RandomUtils.nextLong(0, 31);
        }
    }

    /**
     * 静态工具类生成id
     *
     * @return 获取id
     */
    public static Long generateId() {
        return idUtil.nextId();
    }
}
