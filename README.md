package com.cloudwise.dosm.dbs.trigger;

import com.google.common.collect.Maps;
import lombok.AllArgsConstructor;
import lombok.Getter;

import java.util.Map;

/**
 * @Author frank.zheng
 * @Date 2025-02-13
 */
@Getter
@AllArgsConstructor
public enum CrStatusEnums {

    NEW("New", null),
    OPEN("Open", null),
    REOPEN("Reopen", null),
    APPROVED("Approved", "CR_APPROVED"),
    REJECTED("Rejected", "CR_REJECTED"),
    CLOSED_CANCEL("Closed Cancel", "CR_CLOSED_CANCEL"),
    CLOSED_BACKOUT("Closed Backout", null),
    CLOSED_SUCCESSFUL("Closed Successful", null),
    CLOSED_BACKOUT_FAIL("Closed Backout fail", null),
    CLOSED_ISSUES("Closed Issues", null),
    CLOSED_OVER_RAN("Closed Over Ran", null),
    CLOSED_INCOMPLETE("Closed Incomplete", null),
    ;

    private String label;

    private String notifyScene;


    private static final Map<String, CrStatusEnums> CR_STATIC_MAP = Maps.newHashMap();
    static {
        for(CrStatusEnums item: CrStatusEnums.values()) {
            CR_STATIC_MAP.put(item.getLabel(), item);
        }
    }

    public static CrStatusEnums getByLabel(String label) {
        return CR_STATIC_MAP.get(label);
    }
}
